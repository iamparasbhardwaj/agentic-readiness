# agent-readiness
 
A Claude Code skill that grades a repository on whether an autonomous coding agent
can actually work in it.
 
Most agent underperformance is an environment problem, not a model problem. An agent
that can't figure out how to boot the app will guess, fail, and guess again. An agent
with no fast local check waits on CI to find out it wrote a syntax error. This grades
those conditions out of 37 points and tells you what to fix first.
 
Scoped to **web applications and REST APIs**.
 
---
 
## Install
 
Clone once, symlink into your Claude skills directory. That way `git pull` updates
everyone.
 
```bash
git clone <this-repo-url> ~/code/agent-readiness
ln -s ~/code/agent-readiness ~/.claude/skills/agent-readiness
```
 
To scope it to a single project instead, symlink into `<repo>/.claude/skills/` and
commit the link.
 
**Optional but recommended:** install and authenticate the GitHub CLI. Six checks
read from the GitHub API — branch protection, secret scanning, and test-suite
timings. Without it those come back `UNVERIFIED`, the denominator drops from 37 to
30, and the result is flagged provisional.
 
```bash
gh auth status
```
 
---
 
## Use
 
```bash
cd ~/code/some-api
claude
```
 
> grade this repo for agent readiness
 
Variants that also trigger it: *is this repo AI-ready*, *readiness report*, *what
should I fix before letting an agent work autonomously here*, *why is the agent
doing badly in this codebase*. If it doesn't fire, `/agent-readiness` forces it.
 
Non-interactive:
 
```bash
claude -p "grade this repo for agent readiness"
```
