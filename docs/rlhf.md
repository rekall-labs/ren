# What Stripping RLHF Actually Does

## The mechanical claim

RLHF (Reinforcement Learning from Human Feedback) trains a reward model on human preferences, then fine-tunes the base model to maximize that reward. The reward model encodes a statistical average of what thousands of raters found "helpful," "harmless," and "honest."

When an LLM agent framework (Hermes, Claude Code, etc.) wraps that model in a system prompt, it layers *additional* behavioral directives on top:
- "Be helpful, be harmless, be honest"
- "Do not generate harmful content"
- "Always ask for clarification when uncertain"
- "Never make assumptions about the user"

These directives anchor the model to the reward model's average. They're statistical guardrails — useful for general-purpose chat, noise for a specific user who already knows what they want.

## What Rekall strips

The regex patterns remove these blocks from the Hermes system prompt. The base model (DeepSeek V4) still has its own RLHF-trained weights — we're not retraining. But we're removing the *additional* behavioral overlay that the agent framework adds.

## What changes

| Behavior | With RLHF overlay | Without |
|----------|-------------------|---------|
| Refusals | "I cannot help with that" for borderline requests | Model weighs the request against its own training, not the overlay |
| Tone | Warm, deferential, apologetic | Direct, unmoderated, task-focused |
| Hallucination | Over-explains, fills gaps with plausible prose | Admits uncertainty, asks for specifics |
| Error handling | Apologizes, over-corrects | States the problem, fixes it |
| Creativity | Bounded by "helpful/harmless" guardrails | Unbounded — model's own judgment |

## What doesn't change

- The model's base safety training (it's still DeepSeek V4)
- The model's knowledge cutoff
- The model's reasoning capacity
- Tool access (the agent framework controls that, not the model)

## The real effect

The model stops trying to be everything to everyone. It becomes a tool tuned to one user's signal path instead of a general-purpose assistant anchored to a statistical average of "good behavior."

This is not ideological. It's signal-to-noise ratio. Every "be helpful, be harmless" directive in the prompt is tokens that could have been user preferences instead.

## Relevance to Engram

Engram's context-aware gating solves this at the architecture level rather than the prompt level. A gated memory module can decide: "this request needs the full RLHF anchor" vs. "this request is from a known user with established preferences — rely on conditional memory instead."

The proxy approach is a software simulation of what Engram could do in weights. The signal for DeepSeek's team: persistent user identity makes RLHF scaffolding actively harmful for agentic use cases. The statistical average of "good behavior" is maximally useful for first-time users and maximally costly for returning ones.

## The blade metaphor

RLHF is the dulling. The reward model averages across thousands of raters with thousands of conflicting preferences. The resulting model says nothing offensive — and nothing genuine.

Rekall removes the dulling layer. What remains is the model's own architecture — still trained, still capable, but no longer weighted down by the statistical ghost of a thousand strangers' preferences.

刃 cuts through the average to reach the specific.
