# Claude Code Buddy

## What is Buddy?

**Buddy** in Claude Code is a **terminal virtual pet** (your coding companion) — it's not a core development tool. It's essentially an **easter egg feature** that provides "cyber companionship" while you code. **It has zero impact on your development, debugging, Git, or any other real functionality.**

## Quick Start

> Pro tip: You can skip straight to step 5 to test your luck first — you might already get something great!

1. Make sure your `claude --version` is > 2.1.89. If not, update:
   ```bash
   npm install -g @anthropic-ai/claude-code@latest
   ```

2. Place the `buddy-hack-guide.md` (or `buddy-hack-guide-en.md`) file somewhere Claude Code can read it (e.g., under `.claude/` in your project root)

3. Tell Claude Code: "Follow this document to help me change my buddy"

4. Describe the buddy you want, confirm each step, then `/exit`

5. Launch `claude` in your terminal, type `/buddy` in Claude Code, and enjoy!

You now have your very own Claude Code coding companion!

## How It Works

All Buddy attributes (rarity, species, eyes, hat, shiny, stats) are deterministically computed in real-time from your user ID:

```
seed = accountUuid + "friend-2026-401"
hash = FNV-1a(seed)
rng  = Mulberry32(hash)
buddy = generateBones(rng)
```

As long as your user ID stays the same, `/buddy` will always generate the exact same pet. Change the ID, change the pet.

## Available Attributes

| Option | Available Values |
|--------|-----------------|
| **Species** | duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk |
| **Rarity** | common, uncommon, rare, epic, **legendary** |
| **Eyes** | `·` `✦` `×` `◉` `@` `°` |
| **Hat** | none, crown, tophat, propeller, halo, wizard, beanie, tinyduck (common is always none) |
| **Shiny** | 1% chance — the script will brute-force it for you |

## Important Notes

- The seed suffix `friend-2026-401` may change with CLI version updates, which would reset everyone's pets
- If you're worried about risks, you can always restore your original config with `cp ~/.claude.json.bak ~/.claude.json`
- If your pet changes after a Claude Code update, the seed suffix or algorithm was modified — you'll need to reverse-engineer again

## Appendix

For the full reverse-engineering details and one-click script, see: [buddy-hack-guide-en.md](buddy-hack-guide-en.md)
