# Interview Questions: Foundations

These questions cover Part 1. Each one is phrased the way an interviewer might ask it, usually as a scenario that does not name the topic, because recognizing which concept applies is part of the answer. Each question is followed by what it tests, a model answer, and the follow-ups an interviewer is likely to ask next.

---

## Question 1: Why Did The Bill More Than Double?

> Your agent handles a research task with 3 tool calls, and it costs about ₹2 per task. Product adds a feature that makes the average task need 6 tool calls instead. Your manager expects the cost to roughly double to ₹4, but the bill comes in much higher. Why? And what would you change?

### What Is This Testing?

Whether the candidate knows that the provider bills per token, not per tool call or per user message, and that a stateless model forces the application to resend a growing history on every call. Covered in 1.3 Tokens and 1.5 Context Windows.

### Two Units: Turns And Calls

The answer depends on keeping two units apart:

- **A call** is one request from the application to the model and one response back. The model only ever sees calls.
- **A turn** is one user message plus every call it takes to answer that message. A turn ends when the model gives a final answer, with the stop reason `end_turn`.

A chatbot without tools makes one call per turn, so the two units line up. An agent makes one call per tool round plus one call for the final answer. **For an agent, the number of model calls is not the number of user messages.**

```
Conversation (chat thread)
├── Turn 1: "What's the weather in Mumbai?"
│   ├── Call 1  → tool_use: get_weather("Mumbai")
│   └── Call 2  → "It's 31°C..."                  end_turn
└── Turn 2: "And Delhi?"
    ├── Call 3  → tool_use: get_weather("Delhi")
    └── Call 4  → "Delhi is 24°C..."              end_turn

2 user messages, 4 model calls
```

### What Is Conversation History, Precisely?

History is measured at a specific call, and its scope is set by the application, not the model:

> **For a given model call, the conversation history is every message in the application's stored list for the current conversation, except the current user message. The application decides the scope of "conversation": one task, or one chat thread.**

The stored list holds four kinds of entries: earlier user messages, earlier model replies that answered the user, tool calls, and tool results. A tool call is also a model reply, one where the model asked for a tool instead of answering. A tool result is not written by the model, but it happened in the conversation, so it is history.

Two consequences matter for cost:

- **History grows inside a turn.** Each tool call and its result are added to the list, so the second call of a turn carries more history than the first, even though the user has not said anything new.
- **History is separate from the other prompt components.** The system prompt, tool definitions, retrieved context, and current user message are sent as their own components on every call. They are not copied into the history.

### Model Answer

"The cost does not scale with the number of tool calls. It scales with the total tokens sent across all model calls, and that grows much faster.

The provider bills per token, input and output. The model is stateless, so on every call the application resends every prompt component: the system prompt, the tool definitions, the retrieved context, the current user message, and the conversation history so far. In an agent, one user message takes several model calls, one per tool round plus one for the final answer. After each tool round, the tool call and the tool result are added to the history, and nothing is removed unless the application removes it. So every tool result is paid for again on every later call.

With 3 tool calls, the turn takes 4 model calls. With 6 tool calls, it takes 7, and the later calls carry six tool results each. The number of calls roughly doubles, and the size of each call grows as well, so total input grows much faster than 2x, close to quadratically in the number of steps.

To bring the cost down, I would trim tool results so only what the model needs goes back, summarize or drop old tool results once they have been used, let independent tool calls run in parallel so the turn needs fewer calls, and use prompt caching for the parts of the prompt that repeat on every call."

### The Numbers Behind It

The numbers come from the laptop agent in 1.5 and are invented for illustration. The fixed components add up to 2,450 tokens on every call: system prompt 1,500, tool definitions 500, retrieved budget notes 400, and the user message 50. Each tool round adds 3,050 tokens to the history: the tool call the model wrote, about 50 tokens, and the tool result the application's code returned, about 3,000 tokens. The model wrote only the 50-token call. The 3,000 tokens came from the application running the tool.

```
Each call's input = fixed components (2,450) + history (3,050 × earlier tool rounds)
3 tool calls → 4 model calls
2,450 + 5,500 + 8,550 + 11,600                            = 28,100 input tokens
6 tool calls → 7 model calls
2,450 + 5,500 + 8,550 + 11,600 + 14,650 + 17,700 + 20,750 = 81,200 input tokens
```

Twice the tool calls costs about 2.9 times the input tokens. At ₹2 per task, the manager expected ₹4, and the bill comes in near ₹6. The gap widens as tasks get longer.

The fixed components are paid for on every call too:

```
                     Call 1   Call 2   Call 3   Call 4
System prompt         1,500    1,500    1,500    1,500   same, paid 4 times
Tool definitions        500      500      500      500   same, paid 4 times
Budget notes            400      400      400      400   same, paid 4 times
User message             50       50       50       50   same, paid 4 times
History                   0    3,050    6,100    9,150   tool call 50 + tool result 3,000 per round
Total                 2,450    5,500    8,550   11,600
```

### What Each Fix Targets

- **Trimming tool results** shrinks the 3,050 tokens added per round. That number is added to every later call, so trimming is usually the largest saving.
- **Summarizing or dropping old results** stops the history from growing without limit. Part 2 covers how, including features like auto-compact.
- **Parallel tool calls** reduce the number of calls in a turn. Two independent lookups requested in one response cost one pass of the loop instead of two.
- **Prompt caching** does not shrink the prompt. It makes the repeated part of the prompt cheaper to process on each call. Part 2 covers it.

### Likely Follow-Ups

- **"So is the cost tied to tool calls?"** Only indirectly. A tool call adds tokens to the history and adds another model call to the turn. The bill follows tokens. One exception: some tools run on the provider's side, such as a provider-hosted web search, and can carry a per-use fee on top of tokens. Tools the application runs itself cost nothing extra beyond the tokens their results add.
- **"Besides cost, what else breaks as history grows?"** The context window. Eventually the input no longer fits, which fails partway through a turn, and well before that point, quality drops from context rot.
- **"Is input or output the bigger cost here?"** Input. Each call's output is small, around 50 tokens for a tool call, while the input is resent and grows on every call.