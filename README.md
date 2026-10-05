# Agent Engineering

A ground-up guide to building AI agents, treated as its own discipline rather than a slice of frontend or backend work.

## What Is This?

An agent is a program that uses a language model in a loop: the model decides what to do, a tool performs the action, and the result goes back to the model. This book covers how to make that loop reliable. It starts from the contract between your code and the model, then builds up through context, tools, patterns, testing, and shipping.

No machine learning background is required. The book treats the model as a component with inputs, outputs, and limits, and stays at that level.

## Read It Online

https://akshat1404.github.io/agent-engineering

## What Is Inside?

| Part | Focus |
|---|---|
| 1. Foundations | Prompts, tokens, sampling, context windows, streaming, tool calling |
| 2. Context Engineering | Memory types, retrieval, caching, managing the context window |
| 3. Tool Use | Tool design, schemas, execution and error handling |
| 4. Agent Patterns | ReAct, planning, multi-agent orchestration, human-in-the-loop |
| 5. Evals and Observability | Testing agent behavior, tracing, guardrails and safety |
| 6. Building and Shipping | Architecture, reliability, and a worked case study |

## Status

Work in progress. Parts are written in order, starting with Foundations.

## Build It Locally

The book is built with mdBook, a tool that turns a folder of Markdown files into a website.

1. Install mdBook from its [releases page](https://github.com/rust-lang/mdBook/releases), or run `cargo install mdbook` if you have Rust installed.
2. From the repository root, run `mdbook serve`. This builds the book, serves it on a local address, and rebuilds automatically when a file changes.
3. Open the address it prints, usually `http://localhost:3000`.

## Repository Layout

```
book.toml          mdBook configuration (title, author, theme)
src/SUMMARY.md     table of contents; controls the order of every page
src/README.md      the book's introduction page
src/01-foundations/ ... src/06-building-shipping/    one folder per part
.github/workflows/deploy.yml    builds the book and publishes it to GitHub Pages on every push to main
```

## Feedback

Found an error or something unclear? Open an issue or use the edit link at the top of any page.
