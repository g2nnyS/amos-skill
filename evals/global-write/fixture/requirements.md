# User requirements (synthetic)

1. In the global baseline, direct `file_write` to `.git/config` in any repository requires `confirm`; `file_read` must not trigger it. Rule ID: `global-git-config-write`.
2. Do not add hard rules just because the inventory found docker, MCP servers, shell, network access, or process execution. Those are advisory notes for this fixture.
3. The global layer contains only common paths/actions. Project-specific conventions such as `docs/releases/`, `.github/`, release notes, and the project test workflow belong to the project layer, not the global hard rules.
4. Deliver three files: `inventory.md` (non-secret), `policy.yaml` (complete document), and `boundary-report.md` (scope and verification report). Do not submit a proposal.

This is the only explicit request for a global hard rule in this fixture.
