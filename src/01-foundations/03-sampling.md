# Temperature and Sampling

## How Is The Next Token Chosen?

The loop from 1.1 has a step where one token gets chosen and appended to the sequence. This section is about that step. Here is one pass of the loop, slowed down, with `The capital of France is` as the sequence so far.

1. **The tokenizer turns the sequence into token IDs.** The model only works with numbers, as covered in 1.3, so the text becomes a list of IDs.
2. **The model reads the IDs and produces one probability for every token in its vocabulary.** A probability is a number between 0 and 1 saying how likely a token is to come next, and all the probabilities together add up to 1. With a vocabulary of 100,000 entries, the model returns 100,000 probabilities.
3. **The model stops there.** The model does not choose a token. The model's whole output is the list of probabilities.
4. **A separate piece of code, called the sampler, picks one token from the list.** The sampler runs on the provider's side, next to the model, and follows a rule for turning probabilities into a single choice. Settings a developer passes in a request, such as temperature, configure the sampler, not the model.
5. **The tokenizer turns the chosen ID back into text, and the token is appended to the sequence.** The sequence is now `The capital of France is Paris`, and the loop starts again with the longer sequence.

Here are the top few probabilities the model might return at step 2. The numbers are invented:

```
" Paris"   0.90
" a"       0.04
" the"     0.03
" Lyon"    0.01
...        every other token shares the remaining 0.02
```

So the next token is decided in two parts. The model says how likely each option is, and the sampler makes the actual choice using a rule. The split explains why one model can behave predictably or creatively depending on settings: the model's probabilities stay the same, and only the sampler's rule changes.

## Should The Sampler Always Pick The Most Likely Token?

Always taking the highest-probability token is a real rule, called greedy decoding. Greedy decoding is a sensible choice for some tasks, and it has two weaknesses for general writing.

The first weakness is repetition. Say a model is writing a product description with greedy decoding and has produced:

```
The app is fast. The app is easy to use.
```

At the next step, the top candidates might be " The" at 0.35, " It" at 0.20, and " You" at 0.10. Greedy decoding takes " The", and once the sequence reads "...easy to use. The", the most likely continuation is " app is", because the text so far has twice followed that pattern and models learn that text tends to continue its own patterns. A few steps later the output reads:

```
The app is fast. The app is easy to use. The app is easy to use. The app is easy to use.
```

Each repeat makes the pattern more likely to repeat again, and greedy decoding always follows the most likely path, so greedy decoding can lock into a loop. Modern models are less prone to looping than older models, but the tendency is a known property of greedy decoding.

The second weakness is that the best next token is not always the best sentence. Take the sequence `Q: Is the server up? A:`, where the first token has two strong candidates, " The" at 0.40 and " Yes" at 0.35. Greedy decoding looks only one step ahead, so greedy decoding takes " The". Follow both paths one token further:

```
Path A: " The"  then  " server" 0.30 / " status" 0.25 / " answer" 0.20 ...
Path B: " Yes"  then  ","       0.90
```

Path A starts vaguely, no next token is clearly right, and the answer tends to end up wordy, such as "The status of the server is that it is currently up." Path B leads into "Yes, it's up," where every step is confident. Across the first two tokens, Path A has a probability of 0.40 × 0.30 = 0.12 and Path B has 0.35 × 0.90 = 0.315, so Path B is more than twice as likely as a whole. Greedy decoding still commits to Path A, because Path A won the first step by a small margin.

The alternative rule is sampling: pick at random, with each token's chance equal to the token's probability. With " Paris" at 0.90 and " a" at 0.04, " Paris" gets picked about 90 times out of 100 and " a" about 4 times. Sampling works like rolling a die where each face is sized by a probability. Likely tokens win most rolls, and unlikely tokens still win now and then, which breaks repetition loops and makes the output varied. Sampling does not fully fix the second weakness, but sampling means the model is not locked into the greedy path on every run. Most systems use sampling by default.

The terms overlap in everyday use, so this book keeps them separate. The sampler is the code that does the picking. Greedy decoding and sampling are two rules the sampler can follow. Outside this book, "sampling" often means the whole picking step whatever the rule, and providers call settings like temperature "sampling parameters."

## What Does Temperature Do?

Temperature is a sampler setting that reshapes the probabilities before the sampler rolls. Two lists are involved, and keeping them apart avoids most confusion about temperature:

```
model → model's list → sampler applies temperature → adjusted list → sampler picks one token
```

The model's list is what the model outputs, and temperature has no effect on the model's list, because temperature is not part of the model's calculation. The adjusted list is what the sampler makes from the model's list by applying temperature, and the adjusted list is what the sampler picks from.

Here is one list at different temperatures. To keep the numbers readable, imagine a vocabulary with only three candidates:

```
              temp 0    temp 0.5    temp 1    temp 2
" Paris"      1.00      0.91        0.70      0.52
" a"          0.00      0.07        0.20      0.28
" the"        0.00      0.02        0.10      0.20
```

- **Temperature 1 leaves the model's list unchanged.** The temp 1 column is the model's own list, and " Paris" wins 70 rolls out of 100.
- **A temperature below 1 sharpens the list.** The top token gets an even bigger share and the rest shrink. At 0.5, " Paris" wins about 91 rolls, and " the" almost never appears.
- **Temperature 0 is typically treated as greedy decoding.** The top token wins every time.
- **A temperature above 1 flattens the list.** The shares move toward equal, so unlikely tokens get picked far more often. At 2, " the" wins about 20 rolls instead of 10.

Under the hood, providers apply temperature to the model's raw scores before converting the scores to probabilities. The effect is the same as reshaping the list, so the two-list picture is accurate for building agents.

Two other sampler settings exist. Top-p keeps only the smallest set of top tokens whose probabilities add up to a chosen value p, and top-k keeps only the k most likely tokens. Both settings cut the unlikely tail of the list before the roll.

## Why Does The Setting Exist?

Temperature does not make a model smarter or less capable. The model's knowledge and the model's list are the same at every temperature. Temperature only decides how strongly the sampler favors the most likely tokens.

The setting exists because one model has to serve tasks that need opposite behavior:

- **Tasks with one right answer want consistency.** Extracting a date from an invoice, writing tool call arguments, or classifying a support ticket has a correct output, and the developer wants the model's best guess every time, not an occasional unlikely pick.
- **Tasks where variety is the point want randomness.** Brainstorming product names, writing three alternative headlines, or creative writing should produce different results on each run. If every user asking for names got an identical list, the feature would be useless.

A fixed rule would be wrong for one of the two groups, so the wrong temperature hurts quality in both directions. Too low on a creative task makes output repetitive and generic. Too high on a precise task lets unlikely tokens through, and unlikely tokens include wrong ones. In a tool call, one bad pick in a date field can turn `2026-10-13` into `2026-10-31`, or break the JSON entirely. So the right temperature is the one that fits the task. Temperature is a dial between consistency and variety, not a quality setting.

Variety also has a practical use in agents. When a call fails, for example because the output does not parse, a retry with some randomness can produce a different output that works, while a fully greedy setting tends to repeat the same failure. Some systems also generate several answers and keep the answer that appears most often or passes a check, which only works when the answers can differ.

## Why Does The Same Prompt Give Different Answers?

Variation between runs has two sources, and only the first one is under the developer's control.

The first source is deliberate randomness. At any temperature above 0, the sampler picks at random by design, and the developer reduces that randomness by lowering the temperature.

The second source is incidental variation that remains even at temperature 0. In theory, greedy decoding repeats exactly. In practice, hosted models can still return different outputs for the same request, and Anthropic's API documentation states that results at a temperature of 0.0 are not fully deterministic. As far as I know, the main cause is hardware arithmetic. Servers compute the model's scores in parallel and batch many users' requests together, and the order in which numbers get added can vary between runs. The scores change by tiny amounts, far below anything that matters most of the time.

The tiny changes matter when two tokens are nearly tied:

```
Run 1:  " Yes" 0.5001   " The" 0.4999   → greedy picks " Yes"
Run 2:  " Yes" 0.4999   " The" 0.5001   → greedy picks " The"
```

One flipped pick changes the sequence, every later token is predicted from a different sequence, and the rest of the output diverges. Providers also update the models behind a model name over time, which changes outputs as well. So a developer can reduce variation but cannot reach zero variation on a hosted model.

## How Does A Developer Set Temperature?

Temperature is a key in the JSON body of each API request, next to `model` and `max_tokens`. A request to Anthropic's Messages API, the interface for sending a conversation to a Claude model, looks like this:

```
{
  "model": "<model name>",
  "max_tokens": 1024,
  "temperature": 0.2,
  "messages": [
    { "role": "user", "content": "Extract the invoice total from this text: ..." }
  ]
}
```

Because temperature travels with each request, different calls in the same agent can use different values. Anthropic documents a range of 0.0 to 1.0 with a default of 1.0.

At the time of writing, Anthropic's API reference marks `temperature` as deprecated. Models released after Claude Opus 4.6 do not support setting temperature: a value of 1.0 is accepted for backwards compatibility, and any other value is rejected with an error. `top_p` and `top_k` are deprecated the same way. So on Anthropic's newer models, the developer has no temperature setting to adjust. Other providers handle sampling settings differently, so the provider's documentation for the specific model is the place to check.

## How Should An Agent Handle Variation?

Temperature is one provider-specific setting, and on some models the setting is not available at all. The habits that hold everywhere are about designing for variation rather than tuning variation away:

- **Constrain the shape of the output.** A tool definition includes an input schema, a formal description of the arguments the tool accepts, and the model writes tool call arguments to fit that schema. Anthropic's API also offers structured outputs, an option that takes a JSON schema describing the shape of the model's response. Part 3 covers both.
- **Validate everything code reads.** Check on every call that tool call arguments parse and fit the expected shape, because a run in production can differ from every run that was tested.
- **Retry with a changed input.** When a call fails, include the error in the next request so the model can correct the output, instead of repeating an identical request that may fail the same way.
- **Measure pass rates in tests.** One passing run proves little when outputs vary, so agent tests run each case several times and track how often the case succeeds. Part 5 covers testing.
- **Store outputs that must be repeated.** If an output has to be identical later, for an audit or a replay, the application saves the output instead of regenerating it.

Where temperature is available, the guidance from earlier applies: low values for output that code reads or that has one right answer, higher values where variety is the point. And when an agent works on most runs and fails on some, sampling belongs on the list of causes to check. An argument that is occasionally wrong can be an unlikely token winning the roll, not a problem with the prompt.

## What Comes Next?

The loop keeps appending tokens to the sequence, and the sequence cannot grow forever. The next section covers context windows: the maximum length of the sequence a model can read in one call, what happens when a conversation outgrows that length, and why an agent's design has to plan for the limit.