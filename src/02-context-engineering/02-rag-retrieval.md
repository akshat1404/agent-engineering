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

Searching by meaning starts with an embedding. An embedding model is a separate model from the chat model, and it does one job: it turns a piece of text into a fixed-length list of numbers. **The embedding model is trained so that texts with similar meanings get similar lists of numbers, even when they share no words.**

### Why Is It Called A Vector?

In mathematics, a vector is an ordered list of numbers, and a list of numbers can be read as a position in space. A list of two numbers is a point on a flat plane, with the first number as the horizontal position and the second as the vertical one. A list of three numbers is a point in three-dimensional space. A real embedding with 1,024 numbers is a point in a 1,024-dimensional space, which nobody can picture, but the arithmetic works the same way. So an embedding is a vector, and **texts with similar meanings end up at nearby positions.**

A single number would not be enough, because one number places texts along a single line, so it can express only one property. Suppose every text got one number for "how much is this about time off":

```
"15 days of leave after a child's birth"   → 0.9
"office closed for year-end shutdown"      → 0.9
```

Both are about time off, so both get 0.9, and the search would treat them as the same, even though one is about having a child and the other is about the office calendar. A second number, for "how much is this about family", separates them:

```
                                            time off   family
"15 days of leave after a child's birth"    [0.9,      0.8]
"office closed for year-end shutdown"       [0.9,      0.1]
"maternity health insurance coverage"       [0.2,      0.9]
```

**Every extra number adds another direction in which two texts can differ.** Language has a very large number of independent ways for two texts to differ, which is why real embeddings use hundreds to a few thousand numbers.

The labels "time off" and "family" are invented for illustration. In a real embedding model, nobody designs what each dimension means. The numbers are learned during training, adjusted until texts with similar meanings sit close together, and a concept such as "family" is not stored in one dimension. It is a direction that involves many dimensions at once, and each dimension contributes a little to many concepts. Individual dimensions rarely line up with anything a person could name, and applications never read them. The only thing an application does with embeddings is compare them.

### Is A Vector Made For A Word Or For A Whole Text?

Both exist, for different purposes. Inside a language model, every token is first turned into a vector of its own, and older systems built one vector per word. Those vectors are used inside models, not for search. Retrieval uses text embeddings: an embedding model reads a whole piece of text, such as a sentence, a paragraph, or a page, and produces one vector for the entire text.

**A text embedding has a fixed length, however long the text is.**

```
"paternity leave days"                       → 1,024 numbers
a 500-word section of the Leave Policy       → 1,024 numbers
```

A short passage about one topic gets a vector that represents that topic clearly. A long passage that covers five topics is compressed into the same 1,024 numbers, so its vector mixes all five and matches no single question strongly. This is why documents are split into smaller pieces before they are embedded, covered in the section on chunking.

### How Is Closeness Measured?

Closeness is calculated with a formula, and three are common:

1. **Euclidean distance** is the straight-line distance between two points. A smaller distance means more similar.
2. **Cosine similarity** is the cosine of the angle between two vectors, with each vector drawn as an arrow from the origin. It depends only on the direction each arrow points, not on its length. A value of 1 means the same direction, 0 means unrelated, and -1 means opposite. This is the most common choice for text.
3. **Dot product** multiplies matching numbers and adds the results. It is the core of cosine similarity without the adjustment for length.

Cosine similarity is calculated like this:

```
cosine(a, b) = (a · b) / (|a| × |b|)
a · b = a1×b1 + a2×b2 + ...          the dot product
|a|   = √(a1² + a2² + ...)           the length of vector a
```

Dividing by the two lengths removes length from the result, so only direction is left. Here it is with the two-number vectors from above, where the question "paternity leave days" is `[0.9, 0.85]`, high on both time off and family:

```
Passage "15 days after a child's birth"  [0.9, 0.8]
  dot product = 0.9×0.9 + 0.85×0.8 = 1.49
  lengths     = 1.238 and 1.204
  cosine      = 1.49 / (1.238 × 1.204) = 0.999
Passage "year-end shutdown"              [0.9, 0.1]
  dot product = 0.9×0.9 + 0.85×0.1 = 0.895
  lengths     = 1.238 and 0.906
  cosine      = 0.895 / (1.238 × 0.906) = 0.798
Passage "maternity health coverage"      [0.2, 0.9]
  cosine      = 0.828
```

The passage about leave after a birth ranks first, and the year-end shutdown ranks last, even though it shares "time off" with the question.

Direction is used instead of distance because the length of a vector tends to reflect things such as how long or wordy a text is, while its direction reflects what the text is about. A short passage and a long passage on the same topic can sit at different distances from the origin and still point the same way. Many embedding models also output vectors scaled to a length of 1, and for those, cosine similarity equals the dot product and ranking by distance gives the same order, so in practice the three measures often agree.

### How Does Semantic Search Answer The Question?

Searching by this kind of similarity is called semantic search, meaning search by meaning. It answers the paternity question in three steps:

1. **Ahead of time,** every passage in the HR documents is turned into an embedding and stored. This happens once per passage, not once per question.
2. **At question time,** the employee's question is turned into an embedding.
3. **The question's embedding is compared** with every stored embedding, and the passages are ranked by similarity.

```
Passage A (birth of a child, 15 days)  → 0.89   ranked first
Passage B (submit via HR portal)       → 0.71
Passage C (year-end shutdown)          → 0.42
```

The scores are illustrative. Passage A ranks first without containing the word "paternity", because its meaning is closest to the question.

### Where Is Semantic Search Weak?

Embeddings capture general meaning, which makes them imprecise about exact terms. Suppose the HR documents hold several policies about remote work, and an employee asks:

```
What does policy HR-POL-114 say about remote work?
```

The string `HR-POL-114` carries almost no meaning to an embedding model, so the question's embedding mostly represents "a policy about remote work". HR-POL-109, on equipment for home offices, and HR-POL-121, on remote allowances, look just as close, and the right policy can come back third or not at all. Keyword search handles the same question easily. `HR-POL-114` is a rare string that appears in very few passages, so BM25 gives it heavy weight, and the passages of HR-POL-114 rank first. The same weakness applies to error codes, where `ERR-4012` and `ERR-4021` look nearly identical to an embedding model but name different errors, and to names and product codes.

**Exact identifiers favor keyword search, and paraphrased meaning favors semantic search.** Each method is strong where the other is weak, and real questions often contain both, such as "What does HR-POL-114 say about working from home?", which is why production systems often combine the two.

<!-- Sections still to come: chunking, the indexing and query pipeline, hybrid search and reranking, retrieval as a tool, evaluating retrieval, failure modes. -->