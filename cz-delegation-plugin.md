---
author: Vince Rose
pubDatetime: 2026-04-04T16:00:00.000Z
title: TBD
slug: cz-delegation-plugin
featured: true
draft: true
tags:
  - claude-code
  - ai
  - plugins
description: TBD
---

# Conversation Flow Outline

## 1. The Spark

- I shared the link to [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) — a Claude Code plugin built by OpenAI that adds `/codex:review`, `/codex:adversarial-review`, `/codex:rescue` to delegate work to Codex
- I mentioned I have `cz`/`czd` aliases that point Claude Code at a different API endpoint (z.ai)
- Asked: can we build something similar?
- Claude researched the codex plugin and reported back — it's complex (JSON-RPC, Unix sockets, broker, background jobs, stop gate). Our version would be much simpler.

## 2. Scoping

- I narrowed to three commands: `/cz:review`, `/cz:adversarial-review`, `/cz:ask`
- Cut `/cz:rescue` — "I don't think we really need"
- `/cz:ask` was actually Claude's suggestion, not from codex. I didn't realize until later: "oh you are right idk where i got ask from. i guess you came up with it!"
- Start as personal plugin, think about sharing later

## 3. Design Q&A — One Decision at a Time

Claude walked through the design section by section, asking me to choose at each step:

### Context strategy
- Options presented: A) git diff only, B) git diff + full file contents, C) let cz explore on its own
- I said: "lets go with just copying what the codex plugin does for now. but we can keep track of decisions like this in case we want to diverge later"

### Output handling
- Options: A) raw passthrough, B) current session interprets, C) both
- I asked "what does the codex plugin prefer?" — it uses B for reviews, A for rescue
- Went with B (current session interprets)

### cz vs czd
- Always `cz` (safe mode) — these are read-only operations
- I agreed

### /cz:ask design
- Options: A) pure prompt passthrough, B) prompt + auto-include git diff, C) current session constructs smart context bundle
- Codex doesn't have an ask equivalent — its closest is rescue
- Went with C — current session uses judgment about what context to include

### Packaging
- I asked: "could we put a plugin in the home claude. I like /cz:ask /cz:review etc"
- This pushed us from personal skills to a proper plugin with `/cz:` namespace
- Chose skills-only plugin (approach A) — each SKILL.md instructs the current session

## 4. The "Let cz Explore" Breakthrough

- Claude initially planned to stuff git diffs into the prompt (like codex does)
- I pushed back: "maybe we don't send the full diff. the agent can just explore tasks itself?"
- Claude said `-p` mode is print-only, no tool use
- I corrected: "you are claude, you tell me :P But i'm pretty sure you can invoke claude in a headless mode with an allowed tools list"
- I was right. The invocation became:
  ```bash
  cz -p \
    --allowed-tools "Read,Grep,Glob,Bash(git:*)" \
    --permission-mode dontAsk \
    "Review the uncommitted changes in this repo"
  ```
- Key design win: instead of cramming diffs into a prompt, give `cz` its own tools and let it explore

## 5. Spec + Plan + Implementation

- Claude wrote a design spec and 9-task implementation plan
- I chose subagent-driven development to execute
- Tasks 1-9 executed mostly smoothly (binary wrappers, plugin scaffold, registry, three skills, adversarial prompt template, validation, nix-home wrappers)

## 6. Debugging — Three Bugs in Sequence

First real test of `/cz:review` worked (after a transient API error). But `/cz:adversarial-review` hit three bugs:

### Bug 1: CLAUDE_PLUGIN_ROOT not set
- Skill referenced `${CLAUDE_PLUGIN_ROOT}` to find the adversarial prompt
- Plugin runtime doesn't expose that env var to subprocesses
- Fix: hardcode the path

### Bug 2: Stdin hang
- `cz` process started and just sat there — no output, no error
- It was waiting on stdin even in `-p` mode
- Fix: `< /dev/null` on every invocation

### Bug 3: macOS mktemp
- `mktemp /tmp/cz-adversarial-XXXXX.md` fails on macOS — template must end with X's
- Fix: `mktemp -d` instead

## 7. Using the Plugin to Improve Itself

- Once working, I told Claude: "feel free to also /cz:ask this for an opinion too"
- Used the plugin we just built to get a second opinion on the plugin we just built
- `cz` came back with actionable feedback:
  1. Move ALL system prompts to `prompts/` files (review and ask were still inline, requiring temp files)
  2. Create a wrapper script (`invoke.sh`) to centralize duplicated cz invocation logic across the three skills
  3. Be explicit about telling cz to run `git diff` (don't assume it'll figure it out)
  4. Remove a brittle "two levels up" comment
- We applied all suggestions
- Then ran `/cz:review` on the final state — it found two more housekeeping items (env file guard, orphaned file)
- `cz` called the adversarial prompt template "the strongest piece of the plugin"
