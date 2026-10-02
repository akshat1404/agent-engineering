# Part 1: Foundations

This chapter will cover the very foundations of an agent, we'll start from the something as fundamental as the question : What happens when you write some text to chatGPT and press enter?

## What Happens When You Send Text To A Model?

You type a message and a reply appears. It feels like the model read your message, thought about it, and wrote an answer. The real mechanism is simpler and stranger than that, and everything in this book depends on understanding it. "What happens when you write some text to chatGPT and press enter?" is the modern "What happens when you write a URL in browser and press enter?"

## What Is A Language Model Actually Doing?

A large language model (LLM) is a program trained on a huge amount of text to do one job: given some text, predict what comes next. How it learned to do this is a topic for machine learning engineers, and you don't need it to build agents. What you need is its contract. The input is a sequence of tokens, the small chunks of text the model reads, usually whole words or pieces of words. The output is a probability for every possible next token.

To produce a reply, the system picks one token from those probabilities, adds it to the end of the sequence, and runs the function again on the longer sequence. It repeats until the model produces a special token that means stop. There is no separate stage where the model thinks and then writes. This loop is the whole mechanism.

## Why Start Here?

Because the properties that make agents hard to build all come from this loop. The model sees only what is in the sequence. It produces one token at a time. The sequence has a maximum length. Memory, tools, and planning, which later parts cover, are each a way of working around one of these properties. When an agent misbehaves, the cause is often a misunderstanding at this level.

## What Does This Part Cover?

Six topics, each a consequence of the loop:

- **Prompts:** everything that goes into the input sequence before the model starts predicting.
- **Tokens:** what the sequence is made of, and why that affects cost and speed.
- **Temperature and Sampling:** how one token gets chosen from the probabilities.
- **Context Windows:** the maximum sequence length the model can read in one pass.
- **Streaming:** sending tokens to the caller as they are produced instead of waiting for the full reply.
- **Tool Calling:** how a model that only outputs text can trigger something in the real world.

## What Should You Be Able To Explain Afterward?

Why a long conversation costs more than a short one. Why the same prompt can produce different answers. Why the model does not remember yesterday's conversation. How a text-only model can fetch data or send an email.