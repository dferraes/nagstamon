# Contributing as an AI Agent to Nagstamon

Nagstamon is a cross-platform PyQt6/Qt5 desktop monitoring application (GPL-2.0) that aggregates status from 20+ monitoring systems (Nagios, Icinga, Zabbix, Prometheus, Alertmanager, Centreon, Checkmk, Thruk, etc.). It sits in the systray or floats as a statusbar and shows host/service problems in realtime. This guide helps AI agents understand the codebase structure and development workflows.

## Project Overview & Layout

**Key configuration files:**
- `setup.py` — the only packaging manifest (setuptools; **there is no `pyproject.toml`**). Handles RPM builds (`bdist_rpm`) with per-distro dependency lists (Qt5 for Fedora < 36 / RHEL ≤ 9, Qt6 otherwise), desktop file, manpage and icon installation.
- `build/requirements/{linux,macos,windows}.txt` — pip dependencies per platform (PyQt6, requests, keyring, beautifulsoup4, lxml, arrow, psutil, pysocks, requests-kerberos, requests-ecp, truststore, tzlocal, dbus-python on Linux).
- `.pylintrc` — disables all R, C and W messages; effectively only errors/fatals are enforced.
- `.github/workflows/` — `build-release-latest.yml`, `build-release-stable.yml` (per-platform release builds) and `codeql.yml`.
- `renovate.json` — Renovate configuration (weekly schedule) covering `build/requirements/*.txt` (pip), GitHub Actions and `build/docker/` base images; PyQt6 packages are grouped into one PR. Requires the Renovate GitHub App (or self-hosted runner) to be enabled for the repository.

**Source tree:**
- `nagstamon.py` — entry point (~80 lines). Injects `truststore` into SSL early, acquires a single-instance lock via `lock_config_folder(conf.configdir)`, installs SIGTERM/SIGINT handlers that defer to `statuswindow.exit()` via `QTimer.singleShot` (plus a periodic no-op `QTimer` so Python signal handlers run inside the Qt loop), then starts the Qt app.
- `Nagstamon/config.py` (~1250 lines) — `AppInfo` (NAME, VERSION), OS/desktop detection (`OS`, `OS_WINDOWS`, `OS_MACOS`, `DESKTOP_WAYLAND`, `DESKTOP_NEEDS_FIX`), and the global `conf` singleton holding all settings (servers, filters, notification, colors...).
- `Nagstamon/servers/` — one module per monitoring system plus `Generic.py` (~1800 lines), the base class with HTTP/session handling, status aggregation and actions.
- `Nagstamon/objects.py` — `GenericHost` / `GenericService` domain models that unify data from all server types.
- `Nagstamon/helpers.py` — `STATES` constant, regex filtering, distro detection, config-folder locking, misc utilities.
- `Nagstamon/qui/` — PyQt6 GUI layer: `widgets/` (statuswindow, treeview, systray icon, mediaplayer...), `dialogs/` (server, settings, acknowledge, downtime, actions...), `qt.py` (Qt5/Qt6 abstraction), `dbus.py`.
- `Nagstamon/thirdparty/` — vendored libs: `Xlib`, `ewmh.py` (window management on X11), `sensu_api.py`, `sensugo_api.py`, `zenoss_api.py`.
- `Nagstamon/resources/` — icons, sounds, SVG templates, desktop/appdata/manpage files.
- `build/` — `build.py` (release packaging entry point) plus per-platform assets: `debian/`, `redhat/`, `arch/` (PKGBUILD for local Arch builds), `windows/` (InnoSetup), `macos/` (PyInstaller spec), `docker/`.
- `tests/` — smoke tests + Alertmanager unit tests (see Testing).

**Data flow:** `nagstamon.py` → `Nagstamon.qui` (Qt app, systray, status window) → `Nagstamon.servers` (worker threads fetch status) → results aggregated into `GenericHost`/`GenericService` objects → rendered by the GUI. Configuration flows from the `conf` singleton in `config.py` to all server instances.

## Server Plugin System

Each monitoring system extends `GenericServer` (`Nagstamon/servers/Generic.py`). Registered server types (see imports in `Nagstamon/servers/__init__.py`): Nagios, Centreon, Icinga, IcingaWeb2, IcingaDBWeb(+Notifications), Icinga2API, Multisite (Checkmk), op5Monitor, Opsview, Thruk, Zabbix, ZabbixProblemBased, Livestatus, Zenoss, Monitos3, Monitos4x, SnagView3, Sensu, SensuGo, Prometheus, Alertmanager, LibreNMS.

**Adding a new monitoring system:**

1. Create `Nagstamon/servers/YourServer.py` with a class extending `GenericServer` and a unique `TYPE` class attribute.
2. Import it and call `register_server()` in `Nagstamon/servers/__init__.py`; also hook it into `create_server()` (`Nagstamon/servers/__init__.py:137`).
3. Implement at minimum:
   - `init_http()`: session headers/auth setup (use `self.session` from `GenericServer`).
   - `get_status()`: fetch and parse status; populate `self.hosts` and `self.services` dicts keyed by name with `GenericHost`/`GenericService` objects using standard fields (`name`, `status`, `status_information`, `acknowledged`, `scheduled_downtime`, `flapping`).
   - Optionally `_set_recheck()`, `_acknowledge()`, `_set_downtime()` for actions.
4. Add config widgets in `Nagstamon/qui/dialogs/server.py` `VOLATILE_WIDGETS` (dict of widget → server types; controls which fields are shown/hidden per type).

**Key patterns:**
- Server config is accessed via `conf.servers[self.get_name()]`; defaults are set in the server class `__init__()`.
- Catch exceptions in `get_status()` and return `Result(result=..., error=...)` for graceful degradation. When catching at a higher level (`GenericServer.get_status()`, treeview worker), also set `self.status`/`self.status_description` — otherwise the GUI treats the failure as "everything is OK" (green icon, empty window, no visible error).
- Login forms (IcingaWeb2, IcingaDBWeb, Centreon): do not hardcode expected form fields. Collect all hidden inputs plus the submit button generically — newer Icinga Web 2 versions added a mandatory hidden `uid` field and renamed the submit button to `submit_login`, and some inputs have no `value` attribute (use `.get('value', '')`).
- Some servers proxy to a real implementation after version detection (e.g. `Centreon/__init__.py` switches between `CentreonLegacy` and `CentreonModern`); Alertmanager (`Alertmanager/alertmanagerserver.py`) is a JSON REST API with configurable `map_to_*` fields.
- Logging goes through `self.debug(server=name, debug=msg)`, routed to `debug_queue` when debug mode is on.

## State Values & Severity

`STATES` in `Nagstamon/helpers.py:41` is ordered from best to worst by list index (higher index = worse):

```
UP < UNKNOWN < INFORMATION < WARNING < AVERAGE < HIGH < CRITICAL < DISASTER < UNREACHABLE < DOWN
```

Different monitoring systems use different names; always normalize to this set (e.g. Zabbix "Disaster" → `DISASTER`). Sounds exist only for `STATES_SOUND` = WARNING, CRITICAL, DOWN.

## Build, Run and Test Commands

**Run locally** (needs a display and the dependencies from `build/requirements/linux.txt`):

```bash
python3 nagstamon.py
```

**Tests:**

```bash
python3 -m pytest tests/
# or without pytest:
python3 -m unittest discover -s tests
```

Note: `tests/test_alertmanager.py` requires `pylint` installed (it lints `Nagstamon/servers/Alertmanager` as part of the test). The smoke tests only need the runtime dependencies importable.

**Release builds** are driven by `build/build.py` (run from inside `build/`):

- Windows: PyInstaller → InnoSetup `setup.exe` (optional Authenticode signing when `WIN_SIGNING_CERT_BASE64`/`WIN_SIGNING_PASSWORD` env vars are set). Pass `debug` as first arg for a console build.
- macOS: PyInstaller (`macos/nagstamon.spec`) → `.app` bundle → DMG via `create-dmg`.
- Linux Debian/Ubuntu: copies `build/debian/` to repo root, runs `fakeroot debian/rules build/binary`.
- Linux Fedora/RHEL: `python3 setup.py bdist_rpm`.
- Linux Arch: `makepkg -f` inside `build/arch/` — builds from the local checkout (no tarball), installs via `python setup.py install`; binary lands as `/usr/bin/nagstamon`.

## Testing Strategy

Two test files, both unittest-based (pytest-compatible):

1. **`tests/test_smoke.py`** — dynamically generates one test per `.py` file in `Nagstamon/`:
   - Non-GUI modules (servers, config, helpers, objects, cookies, thirdparty API wrappers): fully imported via `importlib` — catches syntax *and* runtime import errors.
   - GUI modules (`Nagstamon/qui/**`): syntax-checked only via `py_compile` — they need Qt and a display, unavailable in headless CI.
   - Vendored Xlib (`Nagstamon/thirdparty/Xlib`, `ewmh`): syntax-checked only — import can clash with a system python-xlib.
2. **`tests/test_alertmanager.py`** — real unit tests for the Alertmanager server using recorded JSON fixtures (`tests/test_alertmanager_*.json`), plus a pylint run over the Alertmanager package.

There are no GUI-level tests. When changing server status parsing, run the smoke tests and add a fixture-based unit test like the Alertmanager ones if feasible.

## Important Conventions

1. **No circular imports**: GUI modules import servers/config; config and servers must NOT import `Nagstamon.qui` (importing it initializes Qt). Break cycles with local imports if unavoidable.
2. **Thread safety**: servers run in background update threads; status fetching code must not touch GUI objects.
3. **License headers**: every source file carries the GPLv2 header ("Nagstamon - Nagios status monitor for your desktop / Copyright (C) 2008-2026 Henri Wahl..."). Keep it in new files.
4. **Encoding comment**: server modules start with `# encoding: utf-8`.
5. **Single-instance lock**: startup acquires `lock_config_folder(conf.configdir)`; only one instance per config folder may run.
6. **Cross-platform handling**: branch on `OS`, `OS_WINDOWS`, `OS_MACOS` from `Nagstamon.config`; desktop quirks via `DESKTOP_WAYLAND` / `DESKTOP_NEEDS_FIX`. Keyring support is optional and disabled on some KDE+Ubuntu combinations to avoid segfaults.
7. **Style**: 4-space indentation, `snake_case`, string formatting mostly via `format()`/f-strings. Pylint config only enforces errors (R/C/W disabled), so match the surrounding code's idioms rather than inventing new ones.
8. **PyQt6 aborts on unhandled slot exceptions**: any exception escaping a `@Slot()` invoked from C++ (e.g. a worker's `get_status`) kills the process with SIGABRT. Server code must return `Result(result=..., error=...)` instead of raising — `GenericServer.get_status()` and the treeview worker already guard this; keep it that way.
9. **Clean QThread shutdown**: before quitting, clear the worker's `running` flag, emit `finish`, then `quit()`+`wait()` the thread (pattern: `StatusWindow.exit()`). Qt aborts at interpreter exit if a `QThread` is destroyed while running. SIGTERM/SIGINT are handled in `nagstamon.py` via `QTimer.singleShot` deferral; a periodic no-op `QTimer` keeps Python signal handlers responsive inside the Qt loop.
10. **No native `QFileDialog` on KDE**: do not instantiate `QFileDialog()` eagerly or natively — the KDE platform theme's KFileWidget runs KIO jobs whose completion posts a spurious `QEvent::Quit` that shuts the app down. Use static `QFileDialog.getOpenFileName(..., options=QFileDialog.Option.DontUseNativeDialog)` at the point of use.

## Key Files Reference

- `Nagstamon/config.py` — `AppInfo`, OS detection, `conf` singleton, resources paths
- `Nagstamon/servers/Generic.py` — base server: HTTP, auth, TLS, filtering, actions, status aggregation
- `Nagstamon/servers/__init__.py` — plugin registration (`SERVER_TYPES`, `register_server()`), `create_server()`, aggregation helpers (`get_worst_status()`, `get_status_count()`)
- `Nagstamon/objects.py` — `GenericHost` / `GenericService` models
- `Nagstamon/helpers.py` — `STATES`, filters, `lock_config_folder()`, `get_distro()`
- `Nagstamon/qui/__init__.py` — Qt app initialization, systray setup
- `Nagstamon/qui/dialogs/server.py` — per-server-type settings dialog (`VOLATILE_WIDGETS`)

## Debugging Tips

1. Enable debug mode in the settings (or `conf.debug_mode`) to see per-server refresh logs; `debug_to_file` + `debug_file` write them to disk.
2. Config lives in `nagstamon.conf` under the platform config dir (server URLs, credentials — may use the OS keyring). Per-server files live in `<configdir>/servers/server_<name>.conf`.
3. Quick import check for a server module: `python3 -c "import Nagstamon.servers.YourServer; print('OK')"`.
4. GUI debugging requires a running Qt event loop and a display; print/debug-queue logging is more reliable than breakpoints.
5. Test a server end-to-end without the GUI (uses the real config and credentials):
   ```python
   import sys; sys.argv = ['nagstamon']
   from Nagstamon.config import conf
   from Nagstamon.servers import create_server
   srv = create_server(conf.servers['NAME'])
   srv.init_config()          # required - normally called by the treeview worker
   status = srv.get_status()
   print(srv.status, srv.status_code, srv.status_description)
   ```
6. PyQt6 aborts (SIGABRT) carry the Python traceback in the `qFatal` message; find it with gdb: `gdb -batch -ex 'set breakpoint pending on' -ex 'break QMessageLogger::fatal' -ex run -ex 'x/s $rdx' --args python3 nagstamon.py`. Spurious `QEvent::Quit` posters can be traced by breaking on `QCoreApplication::postEvent` with condition `*(short*)($rsi+8) == 20`.

## Security Considerations

- Server credentials are stored in the OS keyring when available (kwallet/SecretService/macOS/Windows backends); plain-text fallback lives in the user config file — never log or commit config files.
- `nagstamon.py` injects `truststore` into SSL early so the OS trust store is used; per-server TLS options (custom CA, ignore verification) exist in config.
- `SECURITY.md` describes the vulnerability reporting process; CI runs CodeQL (`.github/workflows/codeql.yml`).
- Windows release binaries are code-signed only when the signing env vars are present.
