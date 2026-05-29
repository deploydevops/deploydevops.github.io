---
title: "Adopting AI on your engineering team: start with rules, not tools"
date: 2026-05-29
tags: [ai, engineering-management, devops]
read_time: 4
description: "Most AI rollouts buy everyone a license and hope. The ones that stick start with written rules, a small pilot, and metrics that aren't vanity. Here's the playbook."
---

The usual AI rollout goes like this. Buy licenses for the whole team. Send a Slack announcement with a rocket emoji. Declare the company "AI-first." Three months later half the team has quietly stopped using it, and the other half is shipping subtly wrong code faster than before.

The tool was never the hard part. The habit is. Here's the rollout I've watched stick.

## Start with one painful workflow

Don't roll out "AI" to "engineering." Pick one specific, recurring pain and aim there.

Writing the tests nobody wants to write. Debugging the flaky integration suite. First-pass review on Terraform PRs. Onboarding a new hire into a codebase nobody remembers. Pick the thing your team complains about on Fridays, and pilot it with a few volunteers. A narrow win you can point to beats a broad rollout nobody can measure.

## Write the rules before you scale the tool

This is the step everyone skips, and it's the one that decides whether the output is any good.

Put your team's standards in a file the AI reads every time. In Claude Code that's `CLAUDE.md`; other tools read `AGENTS.md`. Check it into the repo. Your conventions, your security rules, your "we don't do it that way" list. [Here's how to wire those rules into hooks and reviewers]({% post_url 2026-05-29-claude-code-for-platform-engineers %}) so they're enforced, not just suggested. Without it, every engineer gets different quality out of the same tool and you can't tell whether the tool is good or just lucky.

```markdown
# Engineering Rules

- Tests for every bug fix. Reproduce first, then fix.
- No new dependencies without a reason in the PR description.
- Match the existing style in the file. Don't reformat code you didn't touch.
- Anything touching auth, billing, or customer data goes to a human.
```

That file is the difference between "the AI writes our code" and "the AI writes code the way we write code."

## Measure something that isn't vanity

"Suggestions accepted" and "lines generated" are vanity metrics. They go up when people accept slop.

Measure whether the painful thing got less painful. Cycle time on the piloted workflow. Defects that escaped to production. How long it takes a new hire to land their first real PR. Pick two or three, take a baseline before the pilot, and look again after a month. Then be willing to say it didn't help, if it didn't. A pilot you can't fail is marketing, not an experiment.

## Keep a human gate on anything that ships

AI output earns the same review a junior engineer's does. No more, no less, and definitely not none.

The fast way to kill trust in the tool is to let one unreviewed AI pull request break prod, or to let [an agent with cloud access get talked into something destructive]({% post_url 2026-05-29-ai-security-for-infrastructure-teams %}). Use the AI for the first pass, even on the review itself, but a human signs off on whatever merges. The goal is a faster path to good code, not a faster path to merge.

## Make the skeptic your lead reviewer

There's always one engineer who thinks the whole thing is a toy. Don't route around them. Hand them the pilot.

The person most determined to find where the AI is wrong is exactly who you want stress-testing it. Their objections become your guardrails: the rules in that `CLAUDE.md`, the items on your human-review list. Win them over with evidence and they turn into your most credible advocate. Skip them and you'll meet their objections later, in production.

## The short version

Adopting AI isn't a purchase. It's a habit change, and habits move one workflow at a time.

Start with one painful thing. Write the rules down. Measure whether it helped. Keep a human on the merge button. The teams that get real value out of this treat the AI like a fast, tireless junior engineer who needs good onboarding and a code review. That's exactly what it is.
