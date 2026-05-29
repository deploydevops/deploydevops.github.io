---
title: "Claude Code for platform engineers: make the AI follow your rules"
date: 2026-05-29
tags: [claude-code, ai, platform-engineering]
read_time: 5
description: "Everyone demos AI writing Terraform. That's the easy part. Here's how to wire Claude Code with rules, hooks, and subagents so it can't ship bad infra."
---

Every "AI for DevOps" demo shows the same thing: type a prompt, watch it generate a Terraform file. Cool. That's also the part I trust the least.

The risk was never that an AI writes Terraform. It's that it writes *plausible* Terraform. A security group that looks fine. An IAM policy with one wildcard you don't notice. And you apply it at 6pm on a Friday because it looked right.

I don't run Claude Code as a code generator. I run it as a junior engineer who has read every one of my rules and can't skip the checks. Here's the setup.

## CLAUDE.md: the rules it reads before every task

`CLAUDE.md` sits in your repo root. Claude Code loads it into context automatically, every session. This is where your standards live, not in your head and not in a wiki nobody opens.

```markdown
# Infrastructure Rules

- Flag any IAM policy with wildcard actions (`s3:*`, `iam:*`) or `Resource: "*"`. Stop and warn before proceeding.
- Never run `terraform apply` on aws-aft-* modules. Those go through the AFT pipeline via PR only.
- Every S3 bucket needs encryption and a public access block. No exceptions.
- Destructive commands (destroy, delete, force-unlock): list the exact resources affected and ask first.
```

That last rule alone has saved me. The model now narrates what it's about to destroy instead of just doing it.

## Hooks: the checks it physically cannot skip

CLAUDE.md is instruction. Hooks are enforcement. A hook is a shell command Claude Code runs at a fixed point in its lifecycle, and the model doesn't get a vote.

A `PreToolUse` hook fires before any tool call. Match it to `Bash` and you inspect every command before it runs. Put this in `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/guard-terraform.sh" }
        ]
      }
    ]
  }
}
```

The hook receives the tool call as JSON on stdin. Read the command, decide, and exit `2` to block it. Anything you write to stderr goes back to Claude as the reason:

```bash
#!/usr/bin/env bash
# .claude/hooks/guard-terraform.sh
command=$(jq -r '.tool_input.command // empty')

case "$command" in
  *"terraform apply"*|*"terraform destroy"*)
    echo "Blocked: run 'terraform plan' and get it reviewed before apply/destroy." >&2
    exit 2
    ;;
esac
exit 0
```

Exit `0` and the command runs. Exit `2` and it's dead, with your message handed to the model so it knows why. This is the difference between "please don't apply" and "you cannot apply."

## Subagents: a security reviewer that isn't you

You will not catch every wildcard at 6pm. A subagent will. Subagents are separate Claude instances with their own context window and their own system prompt. Drop a markdown file in `.claude/agents/`:

```markdown
---
name: tf-security-reviewer
description: Reviews Terraform changes for IAM wildcards, public ingress, and unencrypted resources. Use proactively after editing .tf files.
tools: Read, Grep, Bash
model: sonnet
---

You are an AWS security reviewer. Inspect the changed .tf files and flag:
- IAM policies with wildcard actions or `Resource: "*"`
- Security groups open to 0.0.0.0/0
- Storage or volumes without encryption

Report findings by severity. Review only, never edit files.
```

Because the description says "use proactively," Claude hands off to it after you touch a `.tf` file, without being asked. It reviews in its own context window, so a long review doesn't pollute the main conversation. I keep one of these per concern: security, cost, naming conventions. The security reviewer earns its keep, because [an agent with cloud access is its own attack surface]({% post_url 2026-05-29-ai-security-for-infrastructure-teams %}).

## MCP: give it real tools instead of guesses

The fastest way to get a wrong ARN or a hallucinated API field is to make the model recall it from memory. MCP servers connect Claude Code to real systems so it reads instead of guesses.

```bash
# Connect the GitHub MCP server: real PRs, issues, and code
claude mcp add --transport http github https://api.githubcopilot.com/mcp/ \
  --header "Authorization: Bearer <YOUR_GITHUB_PAT>"
```

Now "open a PR for this module and link the tracking issue" hits the actual API. HashiCorp ships an official Terraform MCP server too, which lets Claude look up real provider and module docs instead of inventing arguments. The rule in my head: if a fact can be looked up, wire up the tool and make it look it up.

## Skills: stop retyping the same five steps

A skill is a saved sequence you trigger with a slash command. Mine for pre-deploy lives in `.claude/skills/pre-deploy/SKILL.md`:

```markdown
---
name: pre-deploy
description: Validate, plan, and security-scan Terraform before any apply
allowed-tools: Read Bash
---

1. Run `terraform validate`.
2. Run `terraform plan -out=tfplan` and summarize the resource changes.
3. Flag any IAM wildcards, public security groups, or unencrypted storage.
4. Stop and report. Do not apply.
```

Type `/pre-deploy` and it runs the whole gate the same way every time. The boring, skippable steps become the un-skippable default.

## The actual point

None of this is about the AI writing code faster. It's about compiling your judgment into the tool so the output clears your bar before you ever see it. CLAUDE.md is the policy, hooks are the enforcement, subagents are the review, MCP is the source of truth, skills are the muscle memory.

Used like that, Claude Code isn't a code generator. It's the second pair of eyes that already read the runbook, doesn't get tired, and doesn't skip the plan step at 6pm on a Friday. That's the part worth automating.
