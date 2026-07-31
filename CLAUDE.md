# CLAUDE.md

Guidance for Claude Code (claude.ai/code) working in this repository.

The canonical contributor guide for AI agents lives in [AGENTS.md](AGENTS.md) —
read it first and keep both files in sync when conventions change. It covers:

- Architecture (config layer → server plugins → PyQt6 GUI) and data flow
- How to add a monitoring system (`Nagstamon/servers/`, `register_server()`)
- Build, run and test commands (`python3 -m pytest tests/`, `build/arch/PKGBUILD`)
- Important conventions, including the Qt6/KF6 pitfalls:
  - PyQt6 aborts the process on unhandled slot exceptions — return `Result(result=..., error=...)` instead of raising in server code
  - QThreads must be stopped cleanly before exit (`running = False` + `finish` + `quit()`/`wait()`)
  - never use the native KDE `QFileDialog` (its KIO jobs post a spurious `QEvent::Quit`) — use static `getOpenFileName(..., DontUseNativeDialog)`
- Renovate (`renovate.json`) handles dependency updates for `build/requirements/*.txt`, GitHub Actions and Docker base images
