# Token cost: commands vs descriptions, Cowork vs Claude Code

Short answer: the surface (Cowork or Claude Code) is not what makes it cheaper. What makes it cheaper is how many tokens move through the model. Both run the same models at the same per-token price.

## What actually drives cost

1. Prompt length. A terse instruction ("List 20 open opportunities in VoxFlow") costs a fraction of a long paragraph describing the same thing. This is the biggest lever you control, and it applies equally in Cowork and Claude Code.
2. Context size. A long conversation or large files in context are re-read on each turn. Shorter sessions cost less.
3. Round trips. Each back-and-forth adds tokens. A single clear command that Claude runs once beats a clarify-then-do exchange.
4. Output size. Ask for compact or JSON output when you do not need prose.

## The real saving

Running the CLI directly in a terminal (you typing `ghl ...`, no model involved) costs zero model tokens. That is the cheapest path for repeatable or bulk work. Use the MCP through Claude when you want the model's help (interpreting, deciding, chaining steps); use the CLI in a terminal when you already know the exact command.

## Practical guidance

- Inside Claude: keep GHL instructions short and specific. Name the subaccount, the object, and the limit. Avoid restating context Claude already has.
- For scheduled or bulk jobs: script the CLI, not a Claude conversation.
- Cowork vs Claude Code: pick by workflow fit, not by an assumed token discount. Neither is inherently cheaper per token; both bill the same model usage.
