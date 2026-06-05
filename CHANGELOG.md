# Changelog

All notable changes to this project are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/) and [Semantic Versioning](https://semver.org/).
Log every update here **before committing** — the changelog entry and the code change go in the same commit (see CLAUDE.md Rule 5).

## [Unreleased]

### Added
- `.claude/settings.json` PreToolUse hook that blocks `git commit` when `CHANGELOG.md` is not staged — Claude Code equivalent of the Kiro `check-changelog` hook [all]

### Changed
- Refined CLAUDE.md guideline: changelog must be updated **before commit** (same commit), and "Adding a New Agent" reworked into "Adding a New Feature" [all]
- Moved `CLAUDE.md` into the IDP project root [all]
