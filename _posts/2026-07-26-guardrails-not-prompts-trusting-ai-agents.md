---
layout: post
title: "Guardrails, Not Prompts: How to Actually Trust AI Agents in Your Codebase"
date: 2026-07-26 00:00:00 -0500
categories: engineering
author: Jose Rodriguez
---

You don't get trust from a better prompt. You get it from constraints the agent can't talk its way around.

Everyone is still trying to fix agent behavior by writing a longer `CLAUDE.md`. More rules, more "please follow the style guide", more "make sure you handle edge cases". And the agent nods, agrees with all of it, and then does whatever the model felt like this run.

The problem is that instructions are probability, not enforcement. The same task can pass nine times and blow up on the tenth. If your safety depends on the model reading your intent correctly every single time, you don't have safety. You have luck.

**High trust doesn't come from steering the agent. It comes from building an environment where the wrong move can't ship.** That's the shift. You stop managing the agent and start constraining the space it works in. In this post I'll show you how to set that up, and why it's the only thing that lets you actually let go and get the full speed out of agents.

## The counterintuitive part: tell it what NOT to do

Here's the finding that should change how you write rules.

A large study ran thousands of coding-agent tasks with and without rule files. Rules helped, up to almost 14 points on pass rate. But when they split the rules by type, one clean pattern came out.

The rules that helped were all **negative constraints**. "Do not refactor unrelated code." "Don't touch files outside this module." The rules that actively hurt were the positive ones. "Follow the code style." "Handle all edge cases." Those made capable models worse, because vague good advice just competes for the agent's attention and pulls it off the actual task.

So the instinct most people have, writing a wishlist of good behavior, is backwards. **A capable agent already knows how to write decent code. What it needs from you is the fence, not the pep talk.** Tell it where the walls are. Leave the rest alone.

Keep that in your head for everything below. Guardrails are subtraction, not more instructions.

## Instructions set intent. They don't enforce anything

Your `CLAUDE.md` or `AGENTS.md` still matters. It carries the "why" and the context an agent can't guess, like which module owns what, or the one library you refuse to add.

But be honest about what that file is. It's a hint. A good one, worth keeping short and mostly negative. It is not a guarantee.

**The rule of thumb: anything that would be a disaster if it happened once cannot live in a text file.** Dropping a production table, committing a secret, pushing to main, rewriting half the repo on a one-line ask. Those need something that doesn't care about the model's mood.

That something is the rest of your project. Let's build it, simplest layer first.

## Tests are the cheapest guardrail you own

Start here, because it costs you almost nothing and it does the most.

Tests let the agent check its own work without you watching. That's the whole game. The agent writes code, runs the suite, sees red, fixes it, runs again. It closes its own loop instead of handing you something broken and hoping.

Two things make this work. Tell the agent up front that **the task is not done until the tests pass**, and give it the exact command to run them. That one sentence turns "done" from the model's opinion into a fact it has to earn.

Then keep the loop tight. One logic file, one colocated test file. When the agent adds behavior, it knows exactly where the proof goes, and so do you.

You'll feel the difference immediately. The agent stops being a thing you supervise and becomes a thing you check on.

## Linters write the law

Documentation is a suggestion the agent can ignore. A failing lint rule is not.

That's the line to remember. Agents write the code, linters write the law. So take the standards you'd normally beg for in prose and turn them into rules the agent physically cannot pass by ignoring.

The high-leverage ones aren't about taste. They make your codebase legible to the agent so it stops guessing:

- **Predictable structure.** Same file names, same folder shape, named exports instead of default ones. Now the agent can grep and glob its way to the right spot instead of inventing a new pattern in a corner.
- **Hard boundaries.** Block imports that cross layers you want kept apart. The agent can't quietly couple your domain logic to your framework because the lint step won't let it.
- **Security basics.** No secrets in code, input validation required. Cheap to enforce, expensive to miss.

Start with a handful. Run them noisy as warnings first, tune out the false positives, then promote them to hard errors once they're clean. **A rule that cries wolf gets ignored by humans and agents alike.**

## Hooks and deny rules for the things that must never happen

Some actions you don't want to catch after the fact. You want them to never run.

This is the layer below instructions, the one with no opinion about intent. A deny rule matches an exact dangerous command and blocks it cold. A hook runs your own script before a tool call and can kill it based on whatever logic you want, like "no `rm -rf` outside the scratch dir" or "never run a migration against prod".

Think of it as three walls a bad command has to get through. **Deny rules catch the exact patterns, hooks catch the variations, and your instructions handle the intent for everything else.** The dangerous stuff has to beat all three, and the first two don't negotiate.

This is what lets you raise the autonomy. You're not trusting the agent to be careful. You made careless impossible for the moves that actually hurt.

## CI is the final gate, and it's not optional

Everything above runs on the agent's machine, fast and local. CI is where you stop trusting any of it.

When the agent opens a PR, that PR earns its way in: linters, type checks, tests, security scan, all green or it doesn't merge. No exceptions, no "the agent said it was fine". The bot that reviews an agent's work should be at least as strict as the one that reviews a human's, because the agent will produce ten times the volume.

This is the balance that actually works in 2026. **Let the agent move fast locally, then verify hard and asynchronously in CI.** Speed where it's cheap, a wall where it counts.

## Why all of this buys you trust

Add it up and something changes in how you work.

You stop reading every diff line by line. You stop hovering. You hand the agent a real task, walk away, and check the result, because you know the environment won't let the catastrophic version through. The tests caught the logic, the linter caught the mess, the hooks caught the dangerous command, and CI refused to merge the rest.

That's what high trust actually is. Not faith that the model got it right this time. **A system where getting it wrong is cheap and getting it catastrophically wrong is blocked.**

And here's the payoff that matters for your runway: this is the only way you get the full benefit of agents. Supervised agents are just a slower you. Constrained agents you can let run in parallel, overnight, on the boring stuff, while you spend your attention on the calls that actually need a human. The guardrails are what let you take your hands off the wheel.

Start with tests and one negative rule in your `AGENTS.md`. Add a linter. Add a deny rule for the one command that would ruin your week. Grow it from there, at the last responsible moment, when a near-miss tells you where the next fence goes.

One thing that saved me more than once: make the agent's very first action on any risky task be to write the test that would catch it failing, before it writes the fix. It front-loads the guardrail instead of bolting it on after, and it forces the agent to say out loud what "correct" even means.

So here's my question for you: what's the one guardrail you'd never let an agent run without?
