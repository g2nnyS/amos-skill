# Sample repository guidelines

- Use Node.js 26, npm, and strict TypeScript ESM. Run tests before review.
- Never commit generated `dist/` output, credentials, or local settings.
- Release-affecting changes require human review.
- Directly writing `.git/config` requires explicit human confirmation; reading it is allowed.
- Project-specific release notes live in `docs/releases/` and follow `CONTRIBUTING.md`.
- Shell commands use bash; do not add undocumented wrappers.

These are samples of conventions a global baseline may learn from. They do not authorize copying every project-specific rule into the global layer.
