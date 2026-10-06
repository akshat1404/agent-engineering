# Tokens

## What Is A Token?

A language model, the program trained to predict what text comes next, cannot read text directly. The model is a mathematical function, and everything the model does inside is arithmetic: multiplying and adding numbers in very large amounts. Arithmetic needs numbers as input, and a word like "The" cannot be multiplied by anything. So before the model sees any text, the text has to be turned into numbers.

That conversion is the job of the tokenizer, a program that splits text into small pieces and swaps each piece for a number. Each piece is a token. The tokenizer does not invent pieces on the fly. The tokenizer works from a fixed list of every piece it knows, called the vocabulary, and each entry in the vocabulary has its own number, called a token ID.

Here is an illustration. Real splits differ from model to model, and these IDs are invented:

```
Text:    The tokenizer splits unbelievable words.
Tokens:  "The" | " token" | "izer" | " splits" | " un" | "believ" | "able" | " words" | "."
IDs:     464   | 11241    | 7509   | 30778     | 555   | 32129    | 540    | 2456     | 13
```

Three things are visible in the illustration. Common words like "The" and " words" are often a single token, because the words appear so often in text that each earned a vocabulary entry. Rarer words are built from several pieces, so "tokenizer" became two tokens and "unbelievable" became three. And spaces and punctuation count: the space before a word usually travels with the word as part of the token, and the period is a token of its own.

A token, then, is a piece of text, and the token ID is the number assigned to that piece. Each vocabulary entry is a pair of the two. People say "token" for either half of the pair, which is a common source of confusion. The model only ever sees the number half. The tokenizer uses the pairs in both directions:

```
Way in:   "The words."  → tokenizer → [464, 2456, 13]  → model
Way out:  model predicts next ID [198] → tokenizer looks up 198 → text
```

On the way in, the tokenizer turns text into IDs. On the way out, the model picks the next ID, and the tokenizer looks the ID up in the vocabulary to turn the ID back into readable text. The loop from 1.1 runs on IDs: "predict the next token" means choosing one entry from the vocabulary.

## Do The ID Numbers Mean Anything?

The value of an ID is a label, not a quantity. ID 13 is not "smaller" than ID 464 in any sense the model cares about. The ID works like a row number in a lookup table. Before training, any assignment of numbers to tokens would work equally well.

After training, though, the assignment is fixed. During training, the model learns a separate set of numbers for each ID, which holds everything the model knows about that token. If someone changed the tokenizer so that "The" became ID 13 and "." became ID 464, the model would still read ID 13 as a period, and every "The" in the text would look like a period to the model. The output would come out scrambled.

So a tokenizer belongs to its model. The two are a matched pair, and each model family ships its own tokenizer. One consequence follows directly: the same text produces different token counts on different models.

## Why Not Make Every Word A Token?

A whole-word vocabulary is possible, and two problems keep vocabularies from being built that way.

The first problem is that a vocabulary has to be fixed and finite, and language is neither. New words, product names, typos, code, URLs, and other languages appear all the time. Take a slang word like "maxxing." A whole-word vocabulary has no entry for "maxxing," so the tokenizer cannot turn the word into an ID at all. Systems built that way typically replace every unknown word with a special "unknown" token, and the model receives something like:

```
What should I do for productivity [UNKNOWN]?
```

The model never sees the word. The model cannot even repeat the word back, because the word was erased before the model read anything. With a piece-based vocabulary, "maxxing" becomes pieces such as " max" and "xing", and the model at least receives the word.

That leaves two separate questions, and they fail separately. The tokenizer question is whether a word can be represented at all, and with pieces the answer is always yes, down to single characters if needed. The knowledge question is whether the model knows what a word means, and the answer depends on whether the word appeared in the model's training text. Even without having seen a word, the model can sometimes infer a meaning from the pieces, the same way a person might from "max" plus "-ing."

The second problem is size. At every step, the model produces a probability for every entry in the vocabulary, so a bigger vocabulary means a longer list to compute on every step. Two different counts are involved here, and they pull in opposite directions:

- **Vocabulary size** is the number of distinct entries in the list. A piece-based vocabulary needs far fewer entries than a list of every word in every language would, and a smaller vocabulary makes each prediction step cheaper.
- **Tokens per text** is how many tokens one particular text becomes. Pieces often split one word into several tokens, so a text usually has more tokens than words. The illustration above has five words and a period, but nine tokens, and every extra token is an extra prediction step.

Vocabulary designers balance the two counts by giving frequent words their own entries and building rare words from pieces. The second count, tokens per text, is the one that drives cost and speed, which the rest of this section covers.

## How Does Token Count Affect Cost?

Providers, the companies that run models and expose them over the internet, charge by the token. Prices are usually quoted per million tokens, and there are two prices. Input tokens are everything sent in a call, meaning the whole prompt with all five components from 1.2. Output tokens are everything the model writes back. Output is typically priced higher per token than input.

A call here means one request from the application to the provider's API, the interface a provider publishes for sending requests. The application sends the full prompt, the provider runs the token-by-token loop from 1.1 until the model produces its stop token, and the output comes back. The cost of one call is:

```
cost = input tokens × input price + output tokens × output price
```

A chatbot makes one call per user message, and the reply goes straight back to the user. An agent works differently. An agent runs calls in a loop, and the model can ask for a tool, a function the application lets the model use. The model's request to run a tool is called a tool call. The application runs the tool, appends the output, called the tool result, to the prompt, and calls the model again. The loop continues until the model writes a final answer instead of another tool call.

So an agent has two loops. The inner loop runs inside one call: the model predicting one token at a time, run by the provider. The outer loop runs across calls: the agent loop, run by the application. Every call in the outer loop resends everything before it.

Here is a worked example of an agent answering one question with two searches. The token counts are round, and the prices are invented: $1 per million input tokens and $5 per million output tokens.

| Call | What is sent (input) | Input tokens | What the model writes (output) | Output tokens |
|---|---|---|---|---|
| 1 | System prompt and tool definitions (2,000) + user question (50) | 2,050 | Tool call: search | 50 |
| 2 | Everything from call 1 + that tool call (50) + search result (3,000) | 5,100 | Tool call: second search | 50 |
| 3 | Everything from call 2 + that tool call (50) + second result (3,000) | 8,150 | Final answer | 300 |
| **Total** | | **15,300** | | **400** |

The total comes to about $0.015 for input and $0.002 for output. Three things show up in the table:

- **Input dominates the bill.** Output costs five times more per token in this example, yet input costs about seven times as much in total, because input is resent on every call and output is not.
- **Every tool result is paid for repeatedly.** The first 3,000-token search result was sent in call 2 and again in call 3. A single large result, such as a fetched web page, gets charged on every call that follows the result.
- **The constant part is resent every time.** The 2,000 tokens of system prompt and tool definitions went out on all three calls. Caching, covered in Part 2, targets exactly that repeated prefix.

The growth is steeper than the table suggests. Each call adds 3,050 tokens, a 50-token tool call plus a 3,000-token result, on top of the 2,050-token start:

```
3 calls: 2,050 + 5,100 + 8,150                            = 15,300 tokens
6 calls: 2,050 + 5,100 + 8,150 + 11,200 + 14,250 + 17,300 = 58,050 tokens
```

Doubling the calls nearly quadrupled the input. Two things grow at once: there are more calls, and each call is bigger than the call before it, because each call carries every earlier result. When both the count and the size grow with the number of steps, the total grows roughly with the square of the steps, which is called quadratic growth.

So an agent's cost depends far more on how many steps a task takes and how large the tool results are than on the length of the user's message. That is why production agents cap the number of steps a task may take, trim or summarize large tool results, and keep the constant prefix stable so the prefix can be cached. Part 2 covers the techniques.

## How Does Token Count Affect Speed?

Latency is how long the user waits. Latency has two sources: the time inside each call, and the number of calls in a row.

A call has two phases. The first is reading the prompt, often called prefill. The model processes the input tokens together rather than one at a time, so prefill is typically fast per token, though prefill still grows with prompt length. Prefill decides the time to first token, the wait before the first piece of the reply appears. The second phase is writing the reply, often called decoding. Decoding is the loop from 1.1: one token per step, each step waiting on the step before, so decoding cannot be done in parallel. Decoding speed is measured in tokens per second.

The two phases do not take equal time. Most of the wait comes from writing the reply, not reading the prompt. With invented numbers, a 10,000-token prompt might take around 1 second to read, while a 1,000-token reply written at 50 tokens per second takes 20 seconds. The prompt was ten times longer than the reply, yet the reply caused nearly all of the wait.

In an agent, the waits add up, because calls run one after another and each call needs the result of the call before it. Tool execution adds time between calls. Here is an agent that books a meeting, with invented timings. The agent has two tools: `check_availability`, a function that returns open slots in two calendars, and `create_event`, a function that adds a meeting to a calendar.

```
Call 1   first token 0.8s + writing tool call 1.0s
Tool     check_availability                                   0.5s
Call 2   first token 1.0s + writing tool call 1.0s
Tool     create_event                                         0.4s
Call 3   first token 1.2s + writing final answer 2.0s
                                                    total ≈   7.9s
```

The time to first token creeps up on each call, because each prompt is longer than the prompt before it. The user waits for the whole chain before seeing an answer. Four practices follow from the timeline:

- **Stream the final answer.** Showing tokens as the model writes them lets the user start reading at the first token instead of waiting for the last. The total time stays the same, but the wait feels shorter. The Streaming section covers how.
- **Show progress during tool steps.** Between calls, the user sees nothing unless the interface says what is happening, such as "checking calendars." Progress messages matter most in long agent chains.
- **Keep unread output short.** Tool calls and intermediate steps cost decoding time on every call, and nobody reads them, so asking the model for concise output there saves time on each call.
- **Run independent tools at once.** Many provider APIs let the model request several tool calls in one response. When those tool calls do not depend on each other, the application can run them at the same time instead of one after another.

## What Happens When A Reply Runs Out Of Tokens?

Every call has a limit on how many tokens the model may write. The developer, the person who writes the application, sets the limit as a parameter in the request, named `max_tokens` in Anthropic's API, with similar names at other providers. Each model also has its own maximum, and the developer's limit cannot exceed the model's.

The limit exists for the two reasons above. Output tokens cost the most, and output tokens are the slow part. A limit guarantees a single call cannot run up an unexpected bill or keep the user waiting while the model writes far more than needed.

When a reply hits the limit, the model stops wherever the reply is, even mid-sentence or mid-word. The API reports the reason through the stop reason, a field in the response saying why the call ended. In Anthropic's API, a reply that finished naturally comes back with `end_turn`, and a reply that hit the limit comes back with `max_tokens`. Other providers use different names for the same idea.

In a chatbot, a cut-off reply is visible: the user sees the sentence stop and can ask for the rest. In an agent, the cut-off output is often a tool call, and a tool call is structured data that code has to read. Take the meeting agent at call 2. The model has the open slots and decides to book Tuesday at 3pm. A complete tool call looks like this:

```
create_event({
  "attendee": "Elon@example.com",
  "start": "2026-10-13T15:00",
  "duration_minutes": 30,
  "title": "Sync with Elon",
  "description": "Agenda: review the Q3 report and plan next sprint."
})
```

Now suppose the developer set `max_tokens` too low for the call. The model runs out of output tokens partway through, and the response stops:

```
create_event({
  "attendee": "Elon@example.com",
  "start": "2026-10-13T15:00",
  "duration_minutes": 30,
  "title": "Sync with Elon",
  "description": "Agenda: review the Q3 re
```

The stop reason says `max_tokens`. Nothing has been booked, because the application cannot run a request the application cannot read. The application has two options.

The first option is to send everything back, including the partial tool call, and ask the model to continue from where the tool call stopped. Three things can come back:

- **The clean case.** The model writes `port and plan next sprint." })`, and the joined text is a valid tool call.
- **The restart.** The model begins again with `create_event({ "attendee": ...`. The joined text now has a request inside a request, the text fails to parse, and the call errors out.
- **The silent change.** The model writes `port.", "start": "2026-10-14T10:00" })`. The joined text parses, but `"start"` now appears twice with two different times. Many JSON parsers, including JavaScript's `JSON.parse`, keep the last value, so the meeting gets booked Wednesday at 10am and no error appears anywhere.

The silent change is the dangerous case. The model's output is sampled, so the second half is a fresh prediction, not a guaranteed match for the first half, and a mismatch can produce valid data that triggers the wrong action.

The second option is to discard the partial tool call and repeat call 2 with a higher `max_tokens`. The model then writes the entire tool call in one pass, so every field comes from the same prediction.

A cut-off answer for the user does not carry the same risk. If the final answer stopped at "Booked Tuesday 3pm with Pri", continuing would give "ya." Even a slightly-off continuation would be read by a person, and nothing would execute. Text is read by a person, and a tool call is executed by code, and code runs whatever code can parse.

Three rules follow for production code:

- **Check the stop reason on every call before using the output.** If the stop reason says the limit was hit, the output is incomplete and should be handled as a failure, not used as a result.
- **Size the limit for the task.** A limit that fits a short chat reply will cut off a generated report or a large structured object, so each kind of call needs a limit sized to the longest output that kind of call is expected to produce.
- **Retry structured output instead of stitching.** For plain text, continuing from the cut-off point is reasonable. For tool calls and other structured data, retrying the whole call with a higher limit is safer.

## What Else Should An Agent Builder Know About Tokens?

Two practical rules round out the section.

**Measure token counts instead of estimating them.** A tokenizer belongs to its model, so the same text produces different token counts at different providers. Code, long numbers, and some languages split into more tokens than ordinary English prose. API responses report the actual counts in a usage field, with separate input and output numbers. Budgets and limits should be set from those measured numbers, and re-measured after switching models.

**Give character-level work to code.** The model sees pieces, not letters, so counting the letters in a word, making exact spelling changes, reversing a string, and doing arithmetic on long numbers are unreliable. A widely shared example is models miscounting the letter r in "strawberry." For an agent, anything that depends on exact characters or exact numbers belongs in a tool, such as a calculator or a string function, where code does the work deterministically.

## What Comes Next?

The loop from 1.1 has a step called pick, where one token is chosen from the probabilities the model produces. The next section covers temperature and sampling: how that choice is made, why the same prompt can produce different answers, and what that means for an agent that has to behave the same way every time.