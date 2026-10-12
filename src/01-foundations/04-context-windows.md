# Context Windows

## What Is A Context Window?

A context window is the maximum number of tokens a model can work with in a single call. A call is one request from the application to the model and one reply back, and a token is a small piece of text, usually a word or part of a word, as covered in 1.3.

Recall the loop from 1.1. The model reads a sequence of tokens, predicts one more token, appends that token to the end, and reads the longer sequence again. The context window is the cap on how long that sequence can get. Anthropic's documentation describes the window as "all the text a language model can reference when generating a response, including the response itself."

That last part matters. The reply is written into the same sequence the model is reading, so the prompt and the reply share one window. Here is a call with an invented, very small window of 1,000 tokens and a prompt of 900 tokens:

```
start:            900 tokens  (the prompt)
after token 1:    901 tokens  (prompt + 1 reply token)
after token 2:    902 tokens  (prompt + 2 reply tokens)
...
after token 100: 1,000 tokens → the window is full, and the model cannot write another token
```

The prompt itself stays at 900 tokens. What grows is the sequence the model reads, which is the prompt plus the reply so far. In this call, the reply can never be longer than 100 tokens, however much the model still had to say. The same window with a 200-token prompt would leave room for an 800-token reply, and a 1,000-token prompt would leave no room at all.

Everything in the prompt counts against the window, not only the conversation. All five components from 1.2 take up space: the system prompt, the tool definitions, the retrieved context, the conversation history, and the current user message. A user can type one short line and still use most of the window, because the rest of the prompt was put there by the application.

## What Counts As Conversation History?

The model is stateless: it keeps nothing between calls. The model has no idea what a conversation is, and it only sees the prompt for the current call. So the application, meaning the code the developer wrote, stores the earlier messages and sends them again with every call. That stored list is the conversation history, and it is the only way the model can respond with earlier messages in mind.

The test for whether something belongs in the history is whether it happened earlier in the conversation. The history holds four kinds of entries:

1. **Earlier user messages.** Every message the user sent before the current one.
2. **Earlier model replies.** Every final answer the model gave the user. A reply becomes history as soon as the conversation moves on.
3. **Tool calls.** Every request the model wrote asking the application to run a function, called a tool. The user never sees tool calls.
4. **Tool results.** Everything the application got back from running those tools. In an agent, tool results are often the largest part of the history.

The other components fail the test. The system prompt and tool definitions are setup the application adds to every call, and nobody said them in the conversation. Retrieved context is fetched for the current question. The current user message is the newest message, not yet answered, and it becomes history once the model responds and the conversation moves on.

A chatbot's history has only the first two kinds. When a user asks a fifth question, the prompt carries the first four questions and their answers as history, and the fifth question as the current message. An agent's history has all four kinds, because the steps the agent took to reach each answer are kept too. Without those steps, the model would not know what it already fetched, and it might ask for the same data again.

Where a conversation starts and ends is also the application's decision, not the model's. The application stores the history as a list, usually under an ID such as a thread or session ID, and the conversation begins when the application creates a new, empty list. Two spans are common:

- **One task.** The list starts empty when the agent receives a request and is discarded when the agent gives its final answer. A nightly job that summarizes a document works this way, with every run starting from no history.
- **One chat thread.** The list lives as long as the thread, so every message, reply, tool call, and tool result stays in it until the user starts a new chat. Chat products typically work this way.

Applications do not keep one endless history across all of a user's chats, because it would overflow any window. Information that should outlast a thread is saved separately and brought back as retrieved context when it is needed, which is memory, covered in Part 2. And even within a thread, the application can drop or summarize older parts of the history when the list gets long. So the precise definition is that the conversation history is the list of earlier messages the application has kept for this task or thread and resends on each call.

## Why Do Agents Run Out Of Room?

Two loops are involved, and both use up the window. The token loop runs inside one call: the model writes one token at a time, filling whatever room the prompt left. The agent loop runs across calls: the application calls the model, the model writes a tool call, the application runs the tool and adds the tool call and its result to the history, and the application calls the model again with the longer prompt. The agent loop repeats until the model writes a final answer instead of another tool call.

Here is an agent with one tool, `fetch_page`, a function that downloads a web page and returns the page's text. A user types:

```
Compare the prices of Laptop A, Laptop B, and Laptop C.
```

Before the first call, the application also loads the user's saved budget notes as retrieved context. The token counts below are invented. Step by step, the first pass of the agent loop goes like this:

1. **The application builds the prompt for call 1.** The history is empty, because nothing has happened yet.

    ```
    System prompt        1,500
    Tool definitions       500
    Budget notes           400
    User message            50
    History                  0
    -------------------------
    Call 1 input         2,450
    ```

2. **Call 1 runs, and the model replies with a tool call.** The model does not answer yet. The model writes a request for the application, `fetch_page("Laptop A price")`, about 50 tokens. This reply is call 1's output, so the reply is not counted in call 1's input.

3. **The application runs the tool.** `fetch_page` downloads the Laptop A product page and returns its text, about 3,000 tokens. The model is not involved in this step. A product page holds far more than the price: menus, specifications, customer reviews, shipping details, and a footer. Unless the application trims the result, all of that text becomes the tool result.

4. **The application adds two entries to the history.**

    ```
    Tool call from step 2              50
    Tool result from step 3         3,000
    ---------------------------------------
    History is now                  3,050
    ```

5. **The application builds the prompt for call 2.** The system prompt, tool definitions, budget notes, and user message are sent again at the same sizes, and the history adds 3,050 tokens, so call 2's input is 2,450 + 3,050 = 5,500 tokens.

The cycle repeats for Laptop B and Laptop C, and each pass adds another 3,050 tokens:

| Component | Call 1 | Call 2 | Call 3 | Call 4 |
|---|---|---|---|---|
| System prompt | 1,500 | 1,500 | 1,500 | 1,500 |
| Tool definitions | 500 | 500 | 500 | 500 |
| Retrieved context (budget notes) | 400 | 400 | 400 | 400 |
| Current user message | 50 | 50 | 50 | 50 |
| Conversation history | 0 | 3,050 | 6,100 | 9,150 |
| **Total input** | **2,450** | **5,500** | **8,550** | **11,600** |

Four rows stay the same on every call. The system prompt, tool definitions, and budget notes describe the setup of each call, and the user message stays the current message for the whole request, because all four calls are answering the same question. Only the history row grows, and nothing leaves the history unless the application removes it. The Laptop A page fetched for call 2 is still being resent on call 4.

Now give the same agent ten laptops to compare and an invented window of 20,000 tokens:

```
Call 1:   2,450 tokens of input
Call 2:   5,500
Call 3:   8,550
Call 4:  11,600
Call 5:  14,650
Call 6:  17,700  → only 2,300 tokens left for the reply
Call 7:  20,750  → the input alone is larger than the window
```

A chat grows differently. In a chat, the history grows by one short user message and one reply per turn, and a person sets the pace. An agent's history grows by whole tool results, several times per user request, with no person in between. Real context windows are much larger than 20,000 tokens, but real tool results can also be much larger than 3,000 tokens, so the same pattern applies.

## What Happens When A Call Does Not Fit?

A call can fail to fit in two different ways, and the API handles the two differently. The behavior below is from Anthropic's documentation at the time of writing, and other providers have their own rules.

**The input alone is too long.** On call 7, the prompt is 20,750 tokens before the model has written anything. When the input alone exceeds the window, the API returns a 400 `invalid_request_error` with the message "prompt is too long," on every model. A 400 error is the HTTP status code meaning the request itself is invalid. Nothing is generated, so an agent hits this error in the middle of a task, with the work half done. A retry cannot help, because sending the same request again produces the same error. The only fix is to make the input smaller.

**The input fits, but the reply runs out of room.** On call 6, the prompt is 17,700 tokens. Suppose the developer set `max_tokens`, the limit on how many tokens the reply may contain, to 4,000. Together that is 21,700 tokens, more than the window. On Claude 4.5 models and newer, the API accepts the request, and the model writes until the sequence reaches the window's limit, here after about 2,300 tokens. The response's stop reason, the field saying why the model stopped writing, is `model_context_window_exceeded`. On earlier models, the API rejects the request up front with a validation error, unless the developer opts in to the newer behavior with a beta header.

The second case looks like the `max_tokens` cut-off from 1.3: the reply stops wherever it is, even mid-word. If the reply was a tool call, the tool call is broken, and the same rule applies. The application treats the output as incomplete and retries the whole call once there is room, instead of asking the model to finish the cut-off half.

Three practices follow for production code:

- **Count before sending.** Anthropic offers a token counting API that measures a request's size before the request is sent, so the application can see that call 7 will not fit before call 7 fails.
- **Check the stop reason on every response.** Two stop reasons now mean the output is incomplete: `max_tokens`, where the developer's limit was hit, and `model_context_window_exceeded`, where the window was hit.
- **Shrink the input before the limit.** When the prompt gets close to the window, the application drops, summarizes, or trims parts of the history. Part 2 covers how.

## Does A Prompt That Fits Get Used Fully?

No. Staying under the limit does not mean the model uses everything in the window equally well. Anthropic's documentation states that "as token count grows, accuracy and recall degrade, a phenomenon known as context rot," where recall means the model's ability to find and use a specific piece of information somewhere in its input. The same page adds that more context is not automatically better.

The laptop agent shows what context rot looks like. By call 4, the prompt holds three full product pages, and the model needs three numbers from all of that text, one price per laptop. Everything else is text the model reads and has to look past, and every extra page adds more. With ten pages, the failures are not errors. They are quieter: two prices mixed up, one laptop missing from the comparison, or an instruction from the system prompt followed less closely because the prompt is now mostly web page text.

Position can matter too. As far as I know, a widely cited 2023 study, "Lost in the Middle," found that models used information near the start or the end of a long input more reliably than information in the middle. Newer models may handle long inputs better, so the finding is a known pattern to watch for, not a fixed rule.

Context rot changes why trimming matters. Shrinking the history is not only about fitting under the limit. A prompt can fit comfortably and still produce worse answers because the prompt is full of text the model does not need. So production agents keep the input lean even when there is room, for example by passing the price instead of the whole page, or by summarizing old tool results.

## What Comes Next?

Part 2 covers the techniques for managing the window: dropping old messages, summarizing them, which is what features like auto-compact do, trimming tool results, and retrieving only the information a call needs. Before that, Part 1 has two more sections. The next section covers streaming: how the tokens of a reply reach the application while the model is still writing them.

Sources: [Context windows, Claude Docs](https://platform.claude.com/docs/en/build-with-claude/context-windows)