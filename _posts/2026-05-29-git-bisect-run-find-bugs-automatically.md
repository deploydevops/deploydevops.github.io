---
title: "git bisect run: find the exact commit that broke your code"
date: 2026-05-29
tags: [git, debugging, automation]
read_time: 5
description: "Stop checking commits one by one. git bisect does a binary search through your history with your test script. 100 commits in 7 steps."
---

Tests are failing. You know they passed a week ago. There are 47 commits between then and now. You could check each one by hand. That sounds miserable.

Or you let git do a binary search.

## How git bisect works

Binary search on your commit history. You tell git which commit is good (tests pass) and which is bad (tests fail). Git checks out the middle commit. You test it. Based on the result, git narrows the range and checks out the next middle commit.

For N commits, it finds the broken one in log2(N) steps. 10 commits takes 3-4 steps. 100 commits takes about 7. 1000 commits takes 10.

## The manual way

```bash
git bisect start
git bisect bad                  # current HEAD is broken
git bisect good abc1234         # this commit was fine

# git checks out the middle commit
# you test it manually, then:
git bisect good   # if this commit passes
# or
git bisect bad    # if this commit fails

# repeat until git finds the first bad commit
```

This works but you're still running tests manually. There's a better way.

## The automated way: git bisect run

Give git a script that returns exit code 0 for "good" and non-zero for "bad". Git runs the whole search without you.

```bash
git bisect start HEAD HEAD~50
git bisect run python3 test_calc.py
```

Git does the rest. It checks out commits, runs your script, and narrows the range based on the exit code.

Here's what it looks like on a 10-commit repo:

```
Bisecting: 4 revisions left to test after this (roughly 2 steps)
[056d906] refactor: simplify discount calculation
running 'python3' 'test_calc.py'
AssertionError: discount failed

Bisecting: 1 revision left to test after this (roughly 1 step)
[1f668a4] ci: add GitHub Actions workflow
running 'python3' 'test_calc.py'
All tests passed

Bisecting: 0 revisions left to test after this (roughly 0 steps)
[02abc73] feat: add logging
running 'python3' 'test_calc.py'
All tests passed

056d906 is the first bad commit
commit 056d906
    refactor: simplify discount calculation
```

3 steps. Found it. Someone changed a minus to a plus.

## Writing the test script

The script needs to return 0 for a good commit and 1 (or any non-zero) for a bad commit. Some special exit codes:

- `125` — skip this commit (can't test it, maybe it doesn't compile)
- `128+` — abort the bisect entirely

For most cases, your existing test suite works directly:

```bash
# Python
git bisect run python3 -m pytest tests/test_pricing.py -x

# Go
git bisect run go test ./pkg/pricing/...

# Node
git bisect run npm test

# Shell script for custom checks
git bisect run bash -c 'make build && ./run_smoke_test.sh'
```

The `-x` flag for pytest (stop on first failure) is useful here. No point running all tests when you only need pass/fail.

## What if commits don't compile?

Sometimes git checks out a commit that doesn't even build. Use a wrapper script:

```bash
#!/bin/bash
# bisect-test.sh

# If it doesn't compile, skip
make build 2>/dev/null || exit 125

# Run the actual test
python3 test_calc.py
```

```bash
git bisect run ./bisect-test.sh
```

Exit code 125 tells git "I can't test this commit, skip it and try another one."

## Real use cases

I've used this on things that have nothing to do with unit tests. A Terraform module that worked last month suddenly fails `terraform plan` — bisect across 30 commits found a variable rename that broke a reference. A GitHub Actions workflow went red after 20 commits — bisect found a permissions change in the checkout step. API response time doubled — I wrote a script that curls the endpoint and checks if it responds under 200ms. Bisect pointed at a commit that added an unindexed query.

Anything with a pass/fail exit code works as the test script.

## Cleanup

When you're done:

```bash
git bisect reset
```

This checks out the branch you were on before you started. Your working tree goes back to normal.

## The cheat sheet

```bash
# Start bisect with known good and bad commits
git bisect start <bad-commit> <good-commit>

# Run automatically with a test script
git bisect run <test-command>

# Check what git found
git show        # shows the first bad commit

# See the diff that broke things
git diff HEAD~1

# Done — go back to your branch
git bisect reset
```

This has been in git since 2005. I don't know why more people don't use it.
