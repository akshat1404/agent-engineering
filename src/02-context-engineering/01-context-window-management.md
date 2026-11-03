# Context Window Management

## Why Does The History Need Managing?

The overview of this part dealt with information an agent can fetch: instead of loading a whole catalog up front, the agent searches for what each call needs. Some tokens cannot be avoided that way. **The conversation history grows on its own while an agent works.** Every tool call and every tool result is added to it, and the whole history is resent on every call, as covered in 1.5.

Here is the laptop agent from 1.5 again. Each round adds a tool call and a product page, about 3,050 tokens, on top of 2,450 tokens of fixed components:

```
Call 1:   2,450 tokens of input
Call 2:   5,500
Call 3:   8,550
...
Call 7:  20,750  → larger than a 20,000-token window
```

Trimming tool results slows the growth, but a long enough task, such as comparing 50 laptops or working through a large codebase, keeps adding rounds until the history reaches the limit. Well before that point, context rot lowers the quality of the answers. This section covers how the application keeps the history within budget while a task is still running.

## What Goes Wrong With A Sliding Window?

The simplest technique is a sliding window: on every call, keep only the most recent N messages of the history and drop everything older.

**A correct sliding window never drops the pinned components.** The system prompt, the tool definitions, and the current user message are pinned, meaning they are included on every call however long the task runs, and the window slides only over the history. This needs care in code, because in the `messages` array sent to the API, the current user message is not stored separately. It sits before the tool rounds of the current turn:

```
[user]       "Compare the prices of Laptop A through Laptop J, and stay within my budget notes."
[assistant]  tool_use: fetch_page("Laptop A")
[user]       tool_result: ...
... 20 more entries ...
```

An implementation that keeps the last 6 items of this array drops the user's request along with the old rounds, and the agent loses its objective.

**Even a correct sliding window loses information the task still needs.** On call 12, with the last 6 messages kept, the model sees the request and the rounds for the last three laptops. The tool calls and results for Laptops A through G are gone. The model then either fetches them again, which costs extra calls and can turn into a loop, or it answers without them or guesses them, which is the missing-context failure from the overview: no error, and a wrong answer.

Parallel tool calls reduce the problem for independent lookups, since ten fetches requested in one response make one round instead of ten. They reduce the number of rounds, not the size of the results. Ten 3,000-token pages fetched in parallel are still 30,000 tokens in the history.

## What Should Be Kept From An Old Tool Result?

The model needed one fact from each product page, the price. So the history does not need the pages. It needs what the model learned from them:

```
Laptop A: $1,500
Laptop B: $2,100
...
```

Ten 3,000-token pages become ten lines of about 10 tokens each, and the final comparison has everything it needs. **The core idea of context window management is to keep the conclusion of a tool result once it has been used, and drop the raw material.**

Someone has to write that conclusion, and there are three common choices:

1. **The application's code, before the result enters the history.** The tool returns `{ "product": "Laptop A", "price": 1500 }` instead of the page. This is the most reliable choice, because code selects the field the same way every time. It only works when the useful field is known in advance.
2. **The application's code, after the result has been used.** The full result stays in the history for the call that needs it, and on later calls the code replaces it with a short placeholder or extract. This is called clearing old tool results. It is simple, but the code decides what to keep without understanding the content.
3. **The model itself.** The model records what it learned as it goes, for example by calling a `save_note` tool. This is the most flexible choice, because the model can judge what matters even in unstructured text. The model can also skip a note or write a wrong one, and every note costs tokens and calls.

The right choice depends on whether the useful part of a result is known in advance:

- **The laptop agent uses choice 1.** Every result is a product page, and the useful fact is always the price, so the tool returns only the price and nothing extra ever enters the history.
- **A research agent uses choice 3.** An agent reading 15 articles to answer "What are the main arguments for and against a four-day work week?" cannot know in advance which points an article contains, so only the model can decide what to keep. The notes work best with a fixed structure, such as `{ "position": "for", "argument": "productivity held steady in trials", "source": "article 4" }`. Once the notes for an article are saved, the application clears the article's full text from the history, combining choice 3 with choice 2. The `source` field lets the model fetch the article again if it later needs a detail it did not save.

## Where Should The Notes Live?

Notes kept in the history are exposed to the same clearing and trimming as everything else. So they live in a scratchpad: storage the application keeps outside the context window, such as a file, a database row, or a key-value entry, scoped to the current task. **Clearing the history never touches the scratchpad, because the scratchpad was never in the history.**

The model can see the notes in two ways:

1. **The application includes them on every call.** The notes go into the context as a pinned block of their own. The model always sees what it has learned, and every call pays for the notes, so they have to stay short.
2. **The model reads them through a tool.** A `read_notes` tool returns the scratchpad when the model asks for it. Calls that do not need the notes stay smaller, but the model has to remember to check.

A common design combines the two: a short summary pinned on every call, such as "9 of 15 articles read, 12 arguments recorded", with the full notes available through the tool.

A scratchpad lasts for one task. Information meant to survive across separate conversations, such as a user's preferences, is memory, covered in the section on memory.

## What Is Compaction?

Tool results are not the only thing in the history. Messages, the model's text between tool calls, and many small tool calls add up too, and a long enough task fills the window even with every tool result trimmed. The last resort is compaction: when the history nears a threshold, the application asks the model to summarize the history so far, and the summary replaces it. Features such as auto-compact in coding agents work this way.

A summary is always shorter than what it replaces, so something is lost. For the research agent halfway through its task, these must survive:

1. **The objective and its constraints.** The original request is usually pinned, but instructions the user added during the task, such as "exclude opinion pieces", live in the history and have to be carried into the summary.
2. **Progress.** Which articles are done and which remain. Without it, the agent reads the same articles again.
3. **Findings and their sources.** The arguments recorded so far, with the article each came from.
4. **Decisions and dead ends.** Such as "article 7 was paywalled, skipped" or "article 11 duplicates article 3". Without them, the agent retries what already failed.
5. **The most recent messages, word for word.** The model is in the middle of a step, so the last few messages stay as they are, and only the older history is summarized.

These are safe to lose: the raw text of articles already turned into notes, the model's working text between calls, the details of past tool calls such as exact arguments and IDs once their results are captured, and duplicated or empty messages.

A compacted history for the research agent can look like this:

```
Task: arguments for and against a four-day work week, balanced, with sources.
User added: exclude opinion pieces.
Progress: 9 of 15 articles read. Remaining: 10 to 15.
Findings: see scratchpad (12 arguments: 7 for, 5 against).
Dead ends: article 7 paywalled, skipped. Article 11 duplicates article 3.
[last 4 messages kept word for word]
```

Two practices make compaction safer:

- **Give the summary a fixed structure.** The compaction prompt asks for specific sections, such as task, progress, findings, and dead ends, instead of "summarize the conversation". A free-form summary is where details go missing.
- **Keep the scratchpad as the source of truth.** Findings already saved in the scratchpad cannot be lost by compaction, and the summary only has to point to them. This is why note-taking comes before compaction.

## When Should The Application Act?

The application needs to know how full the window is before each call. Two sources tell it:

- **The previous response.** Every response from Anthropic's API reports `usage.input_tokens`, the number of input tokens the call used. The next call's input is that number plus whatever was added since.
- **A count before sending.** The token counting API measures a request without running it, at the cost of one extra request.

**The threshold for compaction sits well below the full window, commonly around 70 to 80 percent,** for three reasons:

1. **The reply needs room.** The window holds the prompt and the reply, so space for `max_tokens` has to stay free.
2. **Compaction is itself a model call.** The summary call has to read the history and write the summary, and a full window leaves no room to do either.
3. **Context rot begins before the limit.** Quality already drops as the window fills, so compacting earlier keeps answers better.

With a 200,000-token window, a setup might compact at around 150,000 to 160,000 tokens, leaving room for the summary call and for a full reply.

## How Do The Techniques Fit Together?

The techniques are tried in order, cheapest and most reliable first, because each later one costs more or loses more:

1. **Trim at the source.** Tools return only what the model needs. Nothing extra enters the history, and nothing needs cleaning later.
2. **Reduce the number of rounds.** Independent tool calls go out in parallel, so a task makes fewer calls.
3. **Clear old tool results.** Results that have been used are replaced with short extracts or placeholders.
4. **Take notes in a scratchpad.** When only the model can judge what matters, it saves its findings outside the window.
5. **Compact as a last resort.** When the history still nears the threshold, a structured summary replaces it, with the scratchpad holding what must not be lost.

One more option moves work out of the main history entirely: a separate agent, given its own fresh context, handles a large subtask such as reading all 15 articles and returns only its conclusions. Splitting work across agents is covered in Part 4.

## What Comes Next?

This section managed the history the agent builds during a task. The next section covers the other side: how an agent finds the information a call needs in the first place, by filters and by meaning, so that only the relevant parts enter the context.

Sources: [Context windows, Claude Docs](https://platform.claude.com/docs/en/build-with-claude/context-windows), [Token counting, Claude Docs](https://platform.claude.com/docs/en/build-with-claude/token-counting)