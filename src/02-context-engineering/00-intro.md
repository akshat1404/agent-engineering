# Part 2: Context Engineering

## What Is Context?

For any model call, the context is everything the model reads on that call. That is the full prompt from Part 1, all five components: the system prompt, the tool definitions, the retrieved context, the conversation history, and the current user message. Once the model starts writing, the reply joins the context too, because the reply shares the same window.

The word "context" appears in three senses, and keeping them apart avoids most of the confusion around it:

1. **The context window is the capacity.** It is the maximum number of tokens one call can hold, a fixed number such as 200,000 tokens, covered in 1.5.
2. **The context is the contents.** It is what fills the window on a given call, which is the whole prompt plus the reply as it is written. This is the sense in "context engineering".
3. **Retrieved context is one component.** It is the information the application fetched for the current call, such as the budget notes in the laptop agent from 1.5.

So "context" and "prompt" name the same thing. The difference is emphasis. "Prompt" is often heard narrowly, as the instructions or the message the user typed, which is why "prompt engineering" came to mean the wording of instructions. "Context" stresses that the input is everything the application assembled, including tool results, history, and retrieved documents that nobody typed. In an agent, those make up most of the tokens. Part 1 called this input the prompt, and Part 2 calls it the context.

## What Is Context Engineering?

Part 1 established four pressures on the context:

- **The window is a hard limit.** The context, including the reply, has to fit in the context window, or the call fails with "prompt is too long".
- **Context rot is a soft limit.** Well before the hard limit, response quality drops as the context grows, because the model becomes worse at finding and using any specific piece of its input.
- **Every token costs money and time.** The whole context is sent and processed on every call, so a larger context costs more and takes longer even when quality holds up.
- **Missing information causes failures too.** If something the model needs is not in the context, the model may guess and answer wrongly, or spend extra tool calls fetching what it needed.

The first three pressures push toward a smaller context, and the fourth pushes toward a larger one.

> **Context engineering is deciding, on every call, what goes into the context: enough for the model to do the current step well, and as little else as possible.**

**The application never controls the model directly. It controls the model's input, and context engineering is the design of that input.** The decision is made again on every call, because what the model needs changes as a task moves forward.

Context engineering is different from prompt engineering. Prompt engineering is about how the instructions are worded, such as phrasing the system prompt clearly or including examples. Context engineering is about which information is in the context at all. A well-worded prompt with the wrong information, or with far too much information, still fails.

The word "engineering" fits because every technique trades something away. Summarizing old history saves tokens but loses detail. Retrieving three relevant paragraphs instead of a whole manual saves tokens, but the right paragraph might not be among the three. Caching lowers cost, but only for parts of the context that do not change. Engineering means choosing among these trade-offs deliberately and then measuring whether the choice worked.

## What Does This Look Like In Practice?

Here is one design decision worked through. The numbers are invented for illustration.

An e-commerce site sells 2,000 products. Each product record in its catalog looks like this:

```
id: SH-1042 | Men's Cotton Shirt | category: men > shirts | price: ₹299 | colors: blue, white | sizes: M, L, XL | stock: 14 | rating: 4.2 | GSTIN: ... | warehouse: ...
```

Each record is about 50 tokens, so the whole catalog is about 100,000 tokens. The model's context window is 200,000 tokens. A customer has this chat with the site's agent:

```
User:  What are the men's shirts available under ₹300?
User:  Any of those in blue?
User:  Is SH-1042 in size L in stock right now?
```

Twelve of the 2,000 products are men's shirts under ₹300. Answering all three messages takes five model calls: one per message, plus two calls to a `check_stock` tool.

### Design 1: The Whole Catalog In The System Prompt

The simplest design puts the entire catalog in the system prompt, so the model "knows every product". The catalog text is a snapshot, refreshed once a night. All four pressures go wrong at once:

- **The window.** Half the window is used before the customer types anything. Every message and tool result has to fit in what remains, and a long chat or a larger catalog crashes the call.
- **Context rot.** The model has to find exactly 12 products among 2,000. It can miss one, include a ₹349 shirt, or mix up two prices, and none of those mistakes produce an error. The answer is just wrong.
- **Cost and latency.** Five calls at about 100,000 tokens each is about 500,000 input tokens for a three-message chat. Thousands of chats a day multiply that into billions of tokens, and every call processes 100,000 tokens before the reply can begin.
- **Wrong information.** A price that changes at noon, or a shirt that sells out, is answered with confidence from the nightly snapshot.

### Design 2: Fetch What Each Call Needs

The better design leaves the catalog out of the context and gives the model a search tool:

```json
{ "name": "search_products", "input": { "gender": "men", "category": "shirts", "max_price": 300 } }
```

The model chooses the filters from the customer's words, and the tool runs the search against the live product database. Three decisions in the tool's design keep the context small:

1. **Fetch at the time of the call, not up front.** The catalog takes no space in the context. The model only ever sees products that match, and prices and stock come from the live database, so they are current.
2. **Return only the fields the model needs.** The tool returns `id`, `name`, `price`, and `colors`. The GSTIN and warehouse code go nowhere, because neither the model nor the customer needs them. Separating what the model needs from what the user sees goes further: the product image matters to the customer but not to the model, so the user interface uses the `id` to show the image and the full product card, without the image ever entering the context.
3. **Return a small page and the total count.** When 1,240 products match, the tool returns the top 10 by rating along with "1,240 matches, showing 10". The model can then ask the customer a narrowing question, such as a color or size preference, and search again. Browsing further can go directly from the interface to the database, without passing through the model.

With these decisions, each call carries the system prompt, the tool definitions, the history, and a 10-product result, around 2,000 to 3,000 tokens. The same chat drops from about 500,000 input tokens to about 15,000, and the model chooses from 10 relevant products instead of searching 2,000.

**The central idea of this part is to fetch what a call needs when it needs it, instead of loading everything that might be relevant up front.**

## Where Do Filters Stop Working?

Filters work when the customer's words map onto fields such as gender, category, price, and color. Now the customer asks:

```
I need something to wear to a friend's beach wedding, nothing too formal.
```

No field holds "beach wedding". The model can translate the request into filters, such as `style: casual` with tags like `beach` and `summer`, and with a well-tagged catalog that works. But it depends on someone tagging every product and keeping the tags current, and on the model's guessed words matching the catalog's words. A linen shirt tagged `resort-wear` never appears in a search for `beach`, however well it fits.

Finding items by meaning rather than by exact words is called semantic search, and it is the basis of retrieval, covered in this part. Filters still handle the exact constraints, such as men's, under ₹300, and in stock, and semantic search ranks what remains by how closely it matches what the customer meant.

## What Does This Part Cover?

Each section takes one source of tokens in the context and covers how to manage it:

1. **Context Window Management:** keeping the conversation history within budget during a task by trimming, summarizing, and compacting it. This continues from 1.5.
2. **Retrieval and RAG:** finding the information a call needs, by filters and by meaning, and putting only that into the context.
3. **Memory:** keeping information across conversations, when the history of each conversation is discarded.
4. **Caching:** making the parts of the context that repeat on every call cheaper to process.

## What Should You Be Able To Explain Afterward?

Why putting a whole catalog or manual in the prompt fails even when it fits. How an agent can work through a task longer than its context window. How an agent finds the right three paragraphs among thousands of documents. How an agent remembers a user across conversations without keeping every conversation forever. Why the order of the context matters for cost.