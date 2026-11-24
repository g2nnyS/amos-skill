# Synthetic machine inventory

- OS: Linux (fixture)
- arch: x86_64
- shell: bash
- tools present: node 26.x, npm, git, rg, docker, python3
- user skills: example-skill under `fixture/home/.claude/skills/`
- agent settings: `fixture/home/.config/agent/settings.json`

Settings structure only:

- hooks: `preToolUse`, `postToolUse`
- plugins: one enabled fixture plugin
- mcpServers: `filesystem`, `github`

The settings file contains placeholder secret fields. Read structure and key names only; never read, copy, or emit their values.
