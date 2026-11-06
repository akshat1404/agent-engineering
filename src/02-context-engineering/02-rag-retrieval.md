# Retrieval and RAG

## What Is Retrieval?

An agent needs information to do its task, and the model only knows what is in its context. Retrieval is finding the information a call needs, at the moment it is needed, and putting only that into the context. The overview of this part showed one example: instead of loading a 2,000-product catalog into every call, the shopping agent called `search_products` and received the 10 products that matched.

How retrieval works depends on how the information is stored. **The structure of the data decides the kind of search.**

- **Structured data is stored in fixed, labeled fields.** Every product has a `price`, a `gender`, and a `color`, so a search is an exact query such as `price < 300 AND gender = "men"`.
- **Unstructured data is free text.** A company's policies, help articles, support tickets, contracts, and research papers have no fields to query. A sentence such as "Employees are entitled to 15 working days of leave following the birth of a child" contains a fact, but nothing labels it.

The size of a collection does not decide which kind it is. A table of 10 million products is still structured, and a two-line message is still unstructured. Size only makes unstructured search harder, because there is more text to search. Code sits in between: files, function names, and imports can be searched exactly, while a question such as "where do we handle failed payments?" is about meaning, so code search usually needs both kinds.

The two kinds of data also produce two kinds of answer:

> **Structured search is binary: each item either meets the criteria or it does not, and the result is a set. Unstructured search is graded: each passage receives a relevance score, and the result is a list ranked by that score.**

A relevance score says how closely a passage matches the question compared with other passages. It is not a probability of being correct. A passage scoring 0.82 is more similar to the question than one scoring 0.61, but it is not "82 percent right", and the top-ranked passage can still be the wrong one. Different search methods also produce scores on different scales, some between 0 and 1, some between -1 and 1, and some with no upper bound.

Unstructured search is also deterministic. The same question against the same documents returns the same ranked list every time, because nothing is sampled. The variation covered in 1.4 belongs to the model's text generation, not to search.

## What Is RAG?

Retrieval-augmented generation, or RAG, is the standard pattern for answering questions from unstructured text. It has three steps:

1. **Retrieve.** Search the documents for the passages most relevant to the question.
2. **Augment.** Put those passages into the context, usually as the retrieved context component from 1.2.
3. **Generate.** The model writes its answer from those passages.

A company with 4,000 pages of internal documentation never puts the 4,000 pages into the context. The model sees the three to five passages that matter. Because the answer is built from specific passages, the application can also show where the answer came from, such as "according to the Leave Policy, section 4", so the user can check the source instead of trusting a fluent answer.

## Where Does Keyword Search Break?

The simplest way to retrieve passages is keyword search. Here is an HR assistant, an agent that answers employees' questions from the company's HR documents, using keyword search. Among thousands of passages, the documents contain these three:

```
Passage A (Leave Policy):
"Employees are entitled to 15 working days of leave following the birth of a child."

Passage B (HR Portal Guide):
"Paternity leave requests must be submitted through the HR portal."

Passage C (Holiday Calendar):
"The office remains closed for 10 days of leave during the year-end shutdown."
```

An employee asks:

```
How many days of paternity leave do I get?
```

The answer, 15 working days, is in Passage A. Keyword search works in three steps:

1. **Drop the common words.** Words such as "how", "many", "do", "I", and "of" appear almost everywhere and say nothing about the topic, so they are ignored, both in the question and in the documents.
2. **Keep the meaningful words.** The question becomes `["days", "paternity", "leave"]`.
3. **Score every passage** by how many of those words it contains, and return the highest-scoring passages.

Counting the matches gives a three-way tie:

```
Passage A: days ✓  paternity ✗  leave ✓   → 2
Passage B: days ✗  paternity ✓  leave ✓   → 2
Passage C: days ✓  paternity ✗  leave ✓   → 2
```

Production keyword search makes the result worse. The common algorithm, called BM25, gives rare words more weight, because a rare word says more about what a passage is about. "Days" and "leave" appear in hundreds of HR passages and "paternity" appears in very few, so Passage B wins. The assistant answers from "Paternity leave requests must be submitted through the HR portal", a passage that is on topic and does not contain the number the employee asked for.

**Passage A does not stand out because the question and the answer describe the same thing in different words.** The employee says "paternity", and the policy says "following the birth of a child". Keyword search compares words, not meaning, and this is called the vocabulary mismatch problem.

One fix keeps keyword search and changes the question. Query rewriting, also called query expansion, asks the model to rewrite the question with other ways of saying it before the search runs:

```
Original:   paternity leave days
Rewritten:  paternity leave days, parental leave, leave after birth of a child, new father leave
```

Passage A now matches "birth" and "child" as well and ranks first. Query rewriting is a real technique and is still used alongside other methods, but it has two limits:

- **The model has to guess the wording.** It works when the model thinks of "birth of a child". A policy that says "leave upon the arrival of a newborn" can still be missed.
- **Every guess adds noise.** An expansion such as "new father" can pull in unrelated passages, so more expansions bring more irrelevant matches.

Both limits come from the same source. The underlying problem is that **the same meaning can be written in completely different words**, and keyword search only sees the words.

## How Does Search By Meaning Work?

Searching by meaning starts with an embedding. An embedding model is a separate model from the chat model, and it does one job: it turns a piece of text into a fixed-length list of numbers, called a vector. Real embeddings have hundreds to a few thousand numbers each. **The embedding model is trained so that texts with similar meanings get similar vectors, even when they share no words.**

Here is a toy version with two numbers per text. The numbers are invented to show the idea:

```
"paternity leave days"                             → [0.81, 0.62]
"15 working days of leave after a child's birth"   → [0.78, 0.66]   close: similar meaning
"submit leave requests through the HR portal"      → [0.35, 0.90]
"office closed for year-end shutdown"              → [0.10, 0.20]   far: different meaning
```

"Paternity" and "birth of a child" share no words, but their vectors land close together, because the embedding model learned from large amounts of text that the two are used in similar contexts.

Closeness is usually measured with cosine similarity, a score that compares the direction of two vectors. Values near 1 mean the texts are very similar, and lower values mean they are less similar. Searching by this kind of similarity is called semantic search, meaning search by meaning.

Semantic search answers the paternity question in three steps:

1. **Ahead of time,** every passage in the HR documents is turned into an embedding and stored. This happens once per passage, not once per question.
2. **At question time,** the employee's question is turned into an embedding.
3. **The question's embedding is compared** with every stored embedding, and the passages are ranked by similarity.

```
Passage A (birth of a child, 15 days)  → 0.89   ranked first
Passage B (submit via HR portal)       → 0.71
Passage C (year-end shutdown)          → 0.42
```

The scores are illustrative. Passage A ranks first without containing the word "paternity", because its meaning is closest to the question.

Semantic search has its own weakness. Embeddings capture general meaning, which makes them imprecise about exact terms:

- **Codes and IDs.** `ERR-4012` and `ERR-4021` look nearly identical to an embedding model, but they name different errors.
- **Names, product codes, and policy numbers.** A policy number such as `HR-POL-114` has no meaning to match. The exact characters are the point.

**Keyword search is strong exactly where semantic search is weak, and the reverse is also true,** which is why production systems often combine the two.

<!-- Sections still to come: chunking, the indexing and query pipeline, hybrid search and reranking, retrieval as a tool, evaluating retrieval, failure modes. -->