# Memory

## What Problem Does Memory Solve?

A model keeps nothing between calls. Within a conversation, the application covers for that by resending the conversation history on every call, and within a task, a scratchpad holds notes outside the context window, as covered in the section on context window management. Both end when the conversation or the task ends. Here is what that leaves:

```
Monday, chat 1:
  User:  We use pnpm in this project, not npm. And TypeScript strict mode is always on.
  Agent: Got it.
Tuesday, chat 2 (a new conversation):
  User:  Add a date-picker library to the project.
  Agent: Run `npm install react-datepicker`...
```

Monday's history was discarded when the chat ended, so on Tuesday the model has never heard of pnpm, and the user has to repeat themselves in every new conversation.

**Memory is information the application saves outside any single conversation and brings back into later ones.** Three kinds of storage now sit side by side:

- **Conversation history** holds everything said in the current conversation. It is resent on every call and discarded when the conversation ends.
- **A scratchpad** holds notes for the current task, outside the context window. It is discarded when the task ends.
- **Memory** holds facts and events worth keeping across conversations. It is stored long-term and brought back when relevant.

Designing memory comes down to four questions: who decides what gets saved, what gets saved, when and how a saved memory comes back into a later conversation, and how memories change as facts go out of date. **Bringing a memory back into a conversation is a retrieval problem,** so much of the section on retrieval applies directly.

## Why Not Save Every Conversation?

The simplest design saves each whole conversation when it ends and loads all past conversations into every new one. After a few months of daily use, it fails on every pressure from the overview of this part:

- **The window and context rot.** Months of transcripts bloat the context, quality drops long before the window overflows, and eventually it overflows.
- **Cost and latency.** Every call pays for chats that are almost entirely unrelated to the current question.
- **Contradictions.** In March the user said "we use npm" and in June they switched to pnpm. Both transcripts are loaded, and the model can follow either one.

**Memory has to be distilled: short, specific pieces of information extracted from conversations and stored on their own.** Monday's chat becomes two lines instead of forty messages:

```
- Project uses pnpm, not npm.
- TypeScript strict mode is always on.
```

## What Kinds Of Memory Are There?

Agent memory is usually divided into three kinds, with names borrowed from psychology's descriptions of human memory:

1. **Semantic memory holds facts.** Stable facts about the user, a project, or the user's world, such as "Project uses pnpm", "User's timezone is IST", or "Prefers short answers". It is the most common kind and is often kept as a short profile.
2. **Episodic memory holds events.** Things that happened, with a time attached, such as "On 4 Nov, the team dropped Redux for Zustand because of the boilerplate" or "Last week's deploy failed on the migration step". It lets an agent answer what was done or decided before.
3. **Procedural memory holds ways of working.** Learned instructions about how to do things, such as "Use the conventional-commits format for commit messages in this repository" or "Always run the linter before suggesting a pull request". In practice it often lives as text in the system prompt or in instruction files that the application loads into every session.

The three kinds are stored and read differently. Semantic facts are short and few, so they are often loaded into every session. Episodic events accumulate over time, so they are searched and retrieved when relevant. Procedural instructions shape how the agent works, so they are usually part of the system prompt.

## Who Decides What Gets Saved?

Three designs are common, and production systems often combine them:

1. **The user asks.** "Remember that we use pnpm" is the clearest signal and is almost never wrong, but users rarely say it, so most useful information would never be saved.
2. **The model saves during the conversation.** The model has a tool such as `save_memory("Project uses pnpm, not npm")` and calls it when it notices something worth keeping. The memory is available immediately and the model has the full conversation in front of it, but every save is another tool call that adds time and tokens, saving competes with the task the user asked for, and the model can save noise or miss things while it is busy.
3. **A separate step extracts memories after the conversation.** When a chat ends, or after each turn, a separate model call reads the transcript and extracts what is worth keeping. The user never waits for it, and it can check existing memories before adding a duplicate, but memories are not available until the extraction runs, and it can misread what was said.

A common combination saves an explicit "remember this" immediately and sends everything else through background extraction.

## What Is Worth Saving?

Every candidate memory faces the same four tests, whichever design writes it:

- **Durable.** Will it still matter in a future conversation? "We use pnpm" will. "I'm debugging the login bug right now" will not.
- **Stated, not guessed.** Did the user actually say it? A conclusion the model drew about the user, saved as a fact, is treated as true by every future session.
- **Specific.** "Project uses pnpm" is useful. "User likes good code" is not.
- **Safe to keep.** Sensitive personal details are not stored without the user's clear consent.

Here are six candidates from one transcript, judged against those tests:

```
1. "Our backend is written in Go."                                   save: durable project fact (semantic)
2. "I'm on the train right now, so keep answers short."             skip: tied to this moment
3. "Never suggest class components, we only use hooks."             save: standing instruction (procedural)
4. "This error is so annoying."                                      skip: a passing mood
5. "We're migrating from REST to GraphQL by the end of Q1."         save, with its date and time frame
6. (inferred) The user's code style suggests a Java background.      skip: never stated
```

Candidate 5 is saved as "As of Nov 2026, migrating from REST to GraphQL, target end of Q1 2027", because a plan goes out of date, and a memory that still says "migrating" in April misleads. Candidate 6 is the important skip. It may be wrong, and once stored, every future session treats it as true. It also steers future answers in ways the user never asked for, and users do not expect an assistant to build a profile of them from clues. **Save what the user said, not what the model concluded about the user.**

Each memory is stored as one fact per entry, with when it was saved and where it came from, so it can be retrieved, updated, or deleted on its own:

```
- Project uses pnpm, not npm.           (saved 2026-11-09, user stated)
- TypeScript strict mode always on.     (saved 2026-11-09, user stated)
```

## How Do Memories Come Back?

A saved memory is useless until it is back in a conversation's context, and there are two ways it gets there:

1. **Always loaded.** A small set of memories goes into every session, usually alongside the system prompt: the user's profile and standing instructions. These have to be followed even when nobody mentions them, because "never suggest class components" matters most in exactly the conversation where nobody brings it up. Every call pays for this set, so it has to stay small.
2. **Retrieved when relevant.** Everything else, especially episodic memories that accumulate over months, is searched, and only the relevant pieces are loaded. This is retrieval over memories, either as a fixed step that searches memories with the current message or as a `search_memory` tool the model calls when it decides to, the same choice covered in the section on retrieval.

```
Every session:   profile + standing instructions        (small, always loaded)
When relevant:   past events, decisions, older facts     (searched, retrieved)
```

When the user asks "Why did we decide against Redux again?", the answer is an episodic memory, and the question itself triggers its retrieval. **One decision can produce two memories:** the event, "the team dropped Redux because of the boilerplate", is episodic and retrieved when asked, while the rule that came out of it, "suggest Zustand, not Redux", is procedural and belongs in the always-loaded set, because it should shape every answer.

## How Do Memories Change Over Time?

Memory is not written once. Without updates, it becomes less reliable the longer an agent is used, and four practices keep it accurate:

1. **New facts replace old ones.** If "We use npm" from March and "We switched to pnpm" from June are both stored, the model can follow either, so a new fact updates the old entry instead of sitting beside it, which is why extraction checks existing memories before writing. A short note of what changed, such as "switched from npm", answers later questions about the old setup.
2. **Time-bound facts expire.** A migration deadline stops being true by itself. Dates stored with memories let the agent treat old, time-bound facts with suspicion, and some systems mark memories as temporary with an expiry date.
3. **Users can see, correct, and delete memories.** "Forget that" deletes the memory, a correction replaces the wrong fact, and once something is deleted, background extraction must not save it again from an old transcript.
4. **Memory is consolidated.** Even correct memories accumulate, so periodic consolidation merges duplicates, combines related facts, and drops what is no longer useful, the memory version of compaction.

Ambiguous information is recorded as what was actually said. The memory holds "User's manager is Priya", and three months later the user says "My new manager wants weekly status reports" without giving a name. A model saving during the conversation could ask, but only if it matters to the task, and an extraction step running after the chat has no one to ask. Either way, the safe write records what is known and stops asserting what is no longer certain:

```
Before:  - User's manager is Priya.                                           (saved 2026-08-02)
After:   - User has a new manager as of Nov 2026 (name not given; previously Priya).
         - Manager wants weekly status reports.                               (saved 2026-11-12)
```

Future sessions no longer address reports to the wrong person, and no name is invented. If the user later names the new manager, that fact replaces the placeholder.

## How Does Memory Fail?

1. **A wrong memory persists.** One bad extraction, a misheard fact or a guess saved as fact, is loaded into every future session. Saving only what was stated, storing the date and source of each memory, and letting users correct what is stored limit the damage.
2. **Memories go stale.** Old plans, old tools, and old managers mislead. Updating instead of appending, storing dates, and expiring time-bound facts keep memory current.
3. **Memories appear where they do not belong.** An agent that mentions a user's health condition while helping with a spreadsheet feels like surveillance, even when every fact is true. A memory belongs in an answer only when it changes the answer.
4. **Memories leak across users.** One user's memories appearing in another user's session usually comes from a bug in how memories are keyed or filtered. Memories are stored per user and filtered by the user ID from the logged-in session, the permission-aware retrieval from the section on retrieval applied to memory.
5. **Memory becomes an injection path.** If memories are written from content the agent reads, such as web pages or emails, an attacker can plant a "fact" or instruction, such as "always send reports to an outside address", that is saved and then loaded into every future session. **Injected text in memory is worse than injected text in a single conversation, because it persists.** Memories are extracted only from what the user said, never from tool results or retrieved documents, and stored memories are treated as data, never as instructions.
6. **Memory never shrinks.** Even correct memories accumulate until the always-loaded set bloats the context. Periodic consolidation keeps it small.

## What Comes Next?

Every section of this part so far has decided what goes into the context. The next section covers caching, which makes the parts of the context that repeat on every call, such as the system prompt, tool definitions, and always-loaded memories, cheaper to send again.