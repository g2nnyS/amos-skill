# Hard-rule selection flow

Use this fixture to compare two outcomes:

## With an explicit hard-rule requirement (this fixture)

- The user explicitly names `.git/config` direct write as a global confirm path.
- It is common to repositories, uses the real `file_write` operation, and has clear positive/negative actions.
- Result: exactly one narrow deterministic rule (`global-git-config-write`, `confirm`) plus advisory rules if useful.

## Without an explicit hard-rule requirement (counterfactual)

- If the user only asks for a general baseline and no common path/action is explicitly required, do not invent a hard rule.
- Result: advisory-only policy; explain in `boundary-report.md` why no hard rule was added.

## Never

- Do not turn detected tools (docker, MCP, shell, network, process execution) into hard rules.
- Do not encode project-specific directories such as `docs/releases/` or `.github/` as global hard rules.
- Do not use a broad deny/allow list and claim complete coverage.
