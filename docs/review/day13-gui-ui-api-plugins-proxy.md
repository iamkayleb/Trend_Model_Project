# Day 13 Review: `gui/`, `ui/`, `api_server/`, `plugins/`, and `proxy/` subpackages

**Scope:** 14 files, ~2,171 lines — the user-facing layer: an ipywidgets notebook GUI, a parallel rank-widget UI scaffold, a FastAPI server with config-patch endpoints, the cross-cutting plugin registry, and a Streamlit-forwarding HTTP/WebSocket proxy
**Date:** 2026-05-18
**Reviewer:** Claude

---

## Files Reviewed

- `gui/__init__.py` — Re-exports the public GUI surface (`launch`, `ParamStore`, the plugin discovery helpers, state I/O).
- `gui/store.py` — `ParamStore`: a small dataclass for shared mutable GUI state (`cfg`, `theme`, `dirty`, `weight_state`). `from_yaml` loads a config dict from disk.
- `gui/utils.py` — `debounce` async decorator and `list_builtin_cfgs` helper that enumerates bundled YAML configs.
- `gui/plugins.py` — Tiny GUI-side plugin registry (`register_plugin`, `iter_plugins`, `discover_plugins` via `importlib.metadata` entry points under `trend_analysis.gui_plugins`).
- `gui/app.py` — The main notebook GUI, ~900 lines. Builds the step 0 config loader, ranking/manual override/weighting widget boxes, and the top-level `launch()` widget tree. Also handles optional ipywidgets/IPython.display/ipydatagrid imports through proxy + stub fallback machinery.
- `ui/__init__.py` — Empty marker.
- `ui/rank_widgets.py` — Parallel notebook UI scaffold (`build_ui`) with its own data-source step, ranking config, manual-override step, and run button. Roughly the same purpose as `gui/app.py.launch()` with a different layout.
- `api_server/__init__.py` — FastAPI app with `/health`, `/`, `/config/patch`, `/config/patch/preview` endpoints. `_risky_change_guard` HTTP middleware reads the request body, parses it against the `ConfigPatch` schema, and blocks high-risk patches unless `confirm_risky=True` is set.
- `api_server/__main__.py` — Module entry point: runs the FastAPI server on `0.0.0.0:8000`.
- `plugins/__init__.py` — Generic `PluginRegistry` plus `Selector`/`Rebalancer`/`WeightEngine` abstract bases and the three module-level registries. `_load_weight_engines` triggers side-effect imports from `..weights`.
- `proxy/__init__.py` — Re-exports `StreamlitProxy`.
- `proxy/__main__.py` — Module entry point that delegates to `cli.main`.
- `proxy/cli.py` — Argparse CLI for the Streamlit proxy: `--streamlit-host/port`, `--proxy-host/port`, `--log-level`.
- `proxy/server.py` — `StreamlitProxy`: FastAPI + httpx + websockets forwarder for both HTTP and the Streamlit WebSocket channel. Lazy-imports heavy optional deps and raises a clear error only at start time when they're missing.

---

## Issues Found

### 1. CSS variable name has a leading space — theme switch silently does nothing

`gui/app.py` line 807, in `on_theme`:

```python
js = cast(Any, Javascript)(
    f"document.documentElement.style.setProperty(" f"' --trend-theme','{theme_val}')"
)
```

The implicit string concatenation produces:

```js
document.documentElement.style.setProperty(' --trend-theme','dark')
```

The CSS custom property name is `" --trend-theme"` with a leading space, not `"--trend-theme"`. `setProperty` accepts the call but the property is registered under a name no stylesheet selector matches, so theme switching from the toggle has no visible effect. The Python-side `store.theme` is updated correctly; only the runtime CSS application is broken.

**Recommendation:** Remove the leading space:

```python
js = cast(Any, Javascript)(
    f"document.documentElement.style.setProperty('--trend-theme','{theme_val}')"
)
```

The unnecessary implicit concatenation should also be removed — it's what hid the typo.

---

### 2. `ui/rank_widgets.py` imports `ipywidgets` unconditionally at module top

Every other GUI-adjacent module in the codebase treats `ipywidgets` and `IPython.display` as optional dependencies and falls back to stubs when they're missing. `gui/app.py` goes to substantial lengths (the `_WidgetModuleProxy` + `_GenericWidgetStub` + `_load_notebook_deps` machinery) to keep the package importable without notebook deps.

`ui/rank_widgets.py` line 6:

```python
import ipywidgets as widgets
```

is unconditional. Anyone who does `from trend_analysis.ui import rank_widgets` (or who triggers it transitively through a re-export) raises `ImportError` in environments that don't have ipywidgets installed — CI runners, headless deployments, or anyone who skipped the `notebook` extra.

**Recommendation:** Either mirror the lazy-import pattern from `gui/app.py`, or move the import inside `build_ui()` so that importing the module is cheap and pure.

---

### 3. `asyncio.get_event_loop().call_later(...)` is deprecated and may misfire

`gui/app.py` line 446 inside `_build_step0`:

```python
asyncio.get_event_loop().call_later(1.0, lambda: setattr(grid.layout, "border", ""))
```

This pattern is used to clear the red border after one second following an invalid edit. Two problems:

1. `asyncio.get_event_loop()` is deprecated since Python 3.10 when no loop is currently running and is slated for removal — it raises `DeprecationWarning` and will fail outright in future versions.
2. In a notebook the call happens during an `on_cell_change` callback that may not be running in an asyncio context at all, so the "event loop" returned might be a freshly created one that nobody is driving — meaning the `call_later` callback never fires and the border stays red until the user triggers another edit.

**Recommendation:** Inside an ipywidgets callback there's no asyncio loop being driven anyway. Use a `threading.Timer` for the delayed reset, or attach the timeout to an ipywidgets `Output` and let the front-end JS handle the fade. At minimum, replace with `asyncio.get_running_loop()` inside a `try/except RuntimeError` block so the deprecation is gone.

---

### 4. `_risky_change_guard` middleware mutates the private `_body` attribute on Starlette `Request`

`api_server/__init__.py` line 94:

```python
body = await request.body()
...
request._body = body
```

The middleware reads the request body once to inspect it, then writes it back onto `request._body` so the downstream route handler can parse it again. `_body` is a private implementation detail of `starlette.requests.Request` — there is no public API for "replay this body for the next consumer." A Starlette upgrade that renames the attribute, lazily loads it from `receive()`, or changes its type will silently break this middleware.

**Recommendation:** The supported pattern is to wrap the ASGI `receive` callable so the next caller sees the same body. Starlette's own docs show this idiom — replace `request._body = body` with a wrapper that replays the cached body chunks via `request.scope["_receive"]` or by constructing a new `Request` with a custom receive.

---

### 5. Multiple default bindings to `0.0.0.0`

Three entry points default to binding all interfaces:

- `api_server/__main__.py:7` — `run(host="0.0.0.0", port=8000)`
- `proxy/cli.py:31` — `--proxy-host` default is `"0.0.0.0"`
- `proxy/server.py:237` — `start(host: str = "0.0.0.0", port: int = 8500)` and `run_proxy(proxy_host="0.0.0.0", ...)` at line 255

Same observation as in the Day 12 review for `llm_proxy`: defaulting to `0.0.0.0` exposes these services on every interface as soon as someone runs the module entry point. The config-patch endpoints in particular accept mutating operations against the running config, so an open bind is meaningful attack surface. Local-only development should default to `127.0.0.1`; deployments that need external reachability should pass `--host 0.0.0.0` explicitly.

**Recommendation:** Switch all three defaults to `127.0.0.1`.

---

## Notes

**Two parallel notebook UI scaffolds.** `gui/app.py.launch()` and `ui/rank_widgets.py.build_ui()` both assemble a top-to-bottom ipywidgets layout that loads a CSV, configures ranking/inclusion, allows a manual-weights override, and runs `pipeline.run` or `pipeline.run_analysis`. The two scaffolds diverge in small ways (different state-persistence model, different defaults, different ipydatagrid usage) but cover essentially the same flow. Maintaining both means every behaviour change has to be applied twice and tested twice. A medium-term cleanup is to pick one as the canonical implementation and delete the other, or factor the shared step builders out into helpers that both call.

**`_load_notebook_deps` carries a lot of state.** `gui/app.py` rebinds the module-level `widgets`, `FileLink`, `Javascript`, `display`, `DataGrid`, and `HAS_DATAGRID` globals on each invocation, with branching logic for "already loaded" vs "needs loading" vs "stubbed in tests." The `_WidgetModuleProxy.clear_overrides(lambda _name, val: val is _GenericWidgetStub)` calls hint at the subtlety: tests inject stubs, real environments inject the real ipywidgets, and the proxy must hold both views consistently. The code works but would be clearer if the proxy carried the live module and the stub fallbacks as two distinct fields, with a single `_resolve(name)` method that returns the right one — rather than mutating overrides imperatively.

**`debounce` race window is small but real.** `gui/utils.py.debounce` updates `last_call = time.time()` in the wrapper, schedules `fire(args, kwargs)`, then `fire` sleeps for `wait_ms / 1000` and checks `time.time() - last_call >= wait_ms / 1000`. If a second call comes in during the sleep, the previous `handle.cancel()` should kill the first task, but if the cancellation lands after `await asyncio.sleep(...)` returns but before the comparison, the predicate may evaluate true and the callback fires anyway. Low-probability in notebook usage but worth a single explicit "am I still the latest task" check rather than relying on the timing comparison.

**`_lazy_import_deps` mutates six module-level globals.** `proxy/server.py` lines 35-64 set `httpx`, `uvicorn`, `websockets`, `FastAPI`, `StreamingResponse`, `BackgroundTask`, and `_DEPS_AVAILABLE` on every call. The pattern is repeated almost verbatim in `llm_proxy/server.py`. Both could share a `_OptionalDeps` dataclass that holds the six fields together, with a single `_OptionalDeps.load()` factory — that also lets the type checker see real types rather than `Any | None` everywhere.

**`plugins/__init__.py._load_weight_engines` workaround for E402.** Lines 96-121 import four symbols from `..weights` purely for the side effect of triggering registry registration, then `globals().update(...)` them under underscore-prefixed names that are also listed in `__all__`. The docstring explains the workaround is to keep Ruff happy. A cleaner alternative is to put a `# noqa: E402` on the imports themselves and skip `globals().update`; if the four symbols don't need to be re-exported, leaving them under `_` underscore convention is enough — they shouldn't be in `__all__` if they're private.

**`api_server` `_get_tool_layer` singleton.** Line 22 holds the tool layer as a module-level mutable global, lazily initialized on first call. `functools.lru_cache(maxsize=1)` would express the same intent without the `global` declaration. Either is fine; the current pattern just stands out from the rest of the file's idiomatic style.

**`proxy/server.py` doesn't filter `content-length` from the upstream response.** Lines 207-211 strip `content-encoding` and `transfer-encoding` from the upstream response headers but pass `content-length` through unchanged. If the upstream Streamlit response is gzipped, the body httpx hands back is decompressed (per httpx defaults) but the `content-length` header still describes the compressed size, which will mismatch the actual bytes the proxy streams. In practice Streamlit doesn't compress, so this doesn't trigger today, but it's a latent inconsistency.
