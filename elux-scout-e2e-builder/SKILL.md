---
name: elux-scout-e2e-builder
description: >-
    Builds a complete Scout-Board-driven eLux e2e test end-to-end by combining elux-ui-explorer (find
    elements) and elux-e2e-test-author (POM/pytest conventions) with the extra workflow needed for a test
    that changes a Scout Board / Device configuration setting and verifies its REAL effect on a live
    device: the OU-scoped config-change fixtures, reboot minimization, ground-truth verification one level
    below AT-SPI (X11) when AT-SPI has no signal, extending elux-automation-server and/or the Scout Board
    client when a primitive is missing, working around an undocumented/broken Scout REST endpoint, and
    shipping a branch + PR per touched repo at the end. Use when asked to write a test that changes a
    Scout-managed setting (especially one requiring a device reboot to take effect), verifies something
    AT-SPI can't see, or otherwise needs a new client/server capability -- mirrors how
    test_switch_applications_shortcut.py was built end-to-end in elux-automation-client.
user-invocable: true
---

# eLux Scout-driven e2e test builder (config change -> ground truth -> shipped PRs)

This is the **capstone workflow** for the trickiest kind of eLux e2e test: one that (1) changes a
Scout-Board-managed device setting, (2) needs the device to **reboot** for that setting to actually take
effect, (3) verifies the setting's **real effect** via ground truth that may not be visible through AT-SPI
at all, and (4) may require adding a brand-new primitive to `elux-automation-client` and/or
`elux-automation-server` because neither repo has one yet. It **reuses, rather than replaces**:

- **elux-ui-explorer** -> find any AT-SPI-visible elements the test needs to navigate through (e.g. Device
  configuration > Applications).
- **elux-e2e-test-author** -> the Page Object Model / pytest conventions the test/Page Objects must follow
  (locate by role/name, teardown-first, ground-truth assertions, ticket-linked docstring, steps-only test
  body).

...then adds everything **else** that was needed to ship `tests/test_switch_applications_shortcut.py` in
`elux-automation-client` (read that file alongside this skill -- it's the canonical worked example every
step below points back to).

---

## When you need this skill (vs. the other two alone)

Use this skill when the test needs **any** of the following; if none apply, elux-ui-explorer +
elux-e2e-test-author alone are enough:

- Changing a **Scout Board** config value (not just a Device configuration UI toggle) and verifying it
  actually took effect on the device.
- A setting that only applies after a **reboot**.
- Verifying something a plain XTERM/non-AT-SPI-aware window does (focus, existence, title) -- AT-SPI has
  nothing to check.
- A capability elux-automation-client/elux-automation-server doesn't expose yet.

---

## Step 1 -- Find elements + follow POM conventions (the other two skills)

Do this first, unchanged: use **elux-ui-explorer** to locate any AT-SPI elements you'll click through
(e.g. opening Device configuration), and follow **elux-e2e-test-author**'s conventions for the Page
Object/test shape you're about to write. Everything below is *additional* to those, not a replacement.

---

## Step 2 -- The Scout OU-scoped config-change workflow (reuse it -- don't hand-roll OU/device juggling)

`elux-automation-client` already has this fully built (`fixtures/__init__.py`,
`tests/conftest.py`) -- never write raw `ScoutBoardClient.apply_config()`/`move_device_to_ou()` calls
directly in a test:

```python
def test_my_scout_change(desktop, device_settings, scout_config_change, target_device_name):
    scout_config = ScoutConfig()
    scout_config.desktop_shortcutkeys.keyTaskSwitch = "CTRL_ALT_DOWN"   # whatever section/field
    scout_config_change(
        "my_scout_change",          # <- OU name: MUST match the test's own name (see below)
        scout_config,
        device_name=target_device_name,
        reboot=True,
        wait_online=True,
        reboot_on_teardown=False,   # <- see Step 3
    )
    ...  # act + assert -- the device is already rebooted and ready by the time this line runs
```

- **`target_device_name`** (fixture) resolves the Scout-registered device name from
  `ELUX_SERVER_URL`'s own host via `ScoutBoardClient.find_device_by_ip()` -- **never hardcode a device
  name** in a new test; a device's Scout identity is independent of which one `ELUX_SERVER_URL` happens to
  point at.
- **`scout_config_change`** creates a throwaway OU under the persistent top-level `EluxAutomation` OU,
  disables inheritance, applies your `ScoutConfig`, moves the device in (optionally rebooting +
  waiting), and registers automatic teardown (move back, delete the OU) regardless of test outcome -- no
  `try/finally` needed.
- **Name the OU after the test.** If the task says "name the SubOU the same as the test", pass that exact
  string as the first argument -- e.g. a test named `test_switch_applications_shortcut` uses OU name
  `"switch_applications_shortcut"`.
- Call `scout_config_change(...)` more than once in one test to layer multiple config changes; every
  call's teardown runs automatically, in reverse order.

---

## Step 3 -- Minimize reboots with `reboot_on_teardown`

A config change normally needs **one** reboot to apply it (`reboot=True`) -- but reverting it in teardown
does **not** need a second one if the test only cares that Scout's own OU/config bookkeeping is restored
promptly (the device's *live session* just keeps running with the old setting until it next reboots for
any other reason -- same as any other config change would look, in practice). Pass
`reboot_on_teardown=False` explicitly whenever the test doesn't specifically need the device's *running*
state reset immediately -- it roughly halves total test runtime (~215s -> ~174s, observed live) and, as a
side effect, avoids racing the `desktop` fixture's own afterEach cleanup against an in-flight reboot (a
real bug this exact pattern was found to cause and fix).

---

## Step 4 -- Ground truth beyond AT-SPI (the X11 escape hatch)

elux-e2e-test-author's "assert ground truth, not the widget" principle sometimes needs to go **one level
below AT-SPI entirely**: a plain XTERM window (e.g. from a "Local application" entry) does not register as
its own AT-SPI application and exposes no AT-SPI focus/existence signal. For exactly this case,
`framework.page.Page` already has X11-level (not AT-SPI) primitives -- prefer these over faking a
coordinate or skipping the assertion:

| Need | Use | NOT |
|---|---|---|
| Press a real global hotkey (e.g. a configured "switch applications" shortcut) | `page.keyboard.press("ctrl+alt+Down")` (XTEST, `xdotool key`) | AT-SPI `invoke_action` (no such action exists for a WM-level hotkey) |
| Which window currently has REAL input focus | `page.active_window()` -> `{"window_id", "title"}` | Guessing from AT-SPI's `focused` state (not set for non-AT-SPI-aware windows) |
| Focus a specific window by title | `page.window_action("focus", title)` | Clicking a screen coordinate |
| Does a window with this title exist yet | `page.window_exists(title)` | `time.sleep()` then assuming |

These are real, already-shipped routes/methods (added in elux-automation-server PR #6 /
elux-automation-client PR #14) -- reuse them; don't reinvent.

---

## Step 5 -- If a primitive is genuinely missing, extend `elux-automation-server` (small, 4-file pattern)

When NEITHER AT-SPI NOR an existing X11 primitive covers what you need, add a new one, following the
EXACT pattern used for `keyboard.press()`/`active_window()`:

1. **`server/x11.py`** (elux-automation-server) -- a small function using the existing `_run()` helper
   (e.g. `xdotool ...`), with a docstring explaining WHY this is needed and how it differs from any
   similar existing function.
2. **`server/routes/x11_routes.py`** -- one new Flask route wrapping it, in the standard
   `{"ok": true, "result": ...}` envelope.
3. **`framework/core/server_client.py`** (elux-automation-client) -- one thin `ServerClient` method
   calling the new route.
4. **`framework/page.py`** -- one `Page`/`Keyboard` method calling that, matching the docstring style of
   its neighbors.

**Deploy and smoke-test live BEFORE writing the test that depends on it**:
`elux-automation-server\deploy\transfer_to_box.ps1 -DeviceHost <ip>`, then confirm `/health` still returns
`atspi_connected: true` (the shared `ucdesktop` process didn't crash) and hit the new route directly (a
plain `curl`/`urllib` call) with a harmless value first.

---

## Step 6 -- When the DOCUMENTED Scout REST API doesn't work

Don't assume your payload is wrong -- some Scout Board categories have real server-side gaps:

1. **Check the OpenAPI spec first** (`Scout-API/openapi.json` if available locally) -- confirm your
   field names/enum values/scope match the documented schema exactly.
2. **Reproduce on a throwaway OU** (never the device's real config) -- if a correctly-shaped,
   spec-compliant request still fails (e.g. `"no matching 'useParentEnabled' permission entry exists"`
   even with inheritance disabled), this is a server-side RBAC/permission-schema gap, not your bug.
3. **Only as a last resort**, fall back to the Scout Board **web console's own internal endpoint** for
   that one setting -- capture the real request from a browser's DevTools Network tab while making the
   SAME change through the Scout Board UI, and confirm the exact required fields empirically (e.g. which
   ones are validated vs. merely required-to-be-present -- `set_keyboard_shortcuts_config()` in
   `framework/core/scoutboard/client.py` is the fully-worked template, including how it resolves
   `setupId` dynamically via the documented `GET /api/v1/ou?properties=SetupID` rather than hardcoding a
   value scraped from one browser session).
4. **Gate the workaround transparently inside `apply_scout_ou_config_change()`** (route just that one
   section through the fallback, `ScoutConfig.unstage()` it so the normal path doesn't also attempt it) --
   a test that stages `scout_config.<section>.<field> = ...` should never need to know a workaround exists.

---

## Step 7 -- Post-reboot readiness: don't trust `atspi_connected: true` alone

`ServerClient.wait_until_reachable()` only confirms the HTTP server/AT-SPI *registry* connection is back --
NOT that `ucdesktop`'s own WebKit-rendered UI has actually finished rendering (confirmed live: a session
can even flash back through a splash screen after `/health` first reports healthy). After any post-reboot
health wait, also perform -- and retry -- the REAL first action a caller is about to rely on, not just an
existence probe:

```python
# fixtures._wait_for_desktop_ui_ready() -- the shipped template for this pattern
while True:
    server_client.atspi_click(app, [{"by": "name", "value": "Open start menu"}], 5, 0.2)
    if _wait_until(lambda: server_client.atspi_has_open_overlay(app), timeout=5):
        server_client.atspi_click(app, [...], 5, 0.2)   # close it again
        return
    ...  # retry until an overall deadline (~180s), then raise
```

Reuse `fixtures._wait_for_desktop_ui_ready()` directly rather than re-deriving this.

---

## Step 8 -- Write the test (steps only)

The test body must stay a short arrange/act/assert sequence -- every retry/verification loop above lives
in a Page Object or fixture, never inline in the test:

```python
"""No associated ticket.

<one paragraph: the Scout setting this proves, and the ground truth used to verify it took effect>
"""
from __future__ import annotations
from typing import TYPE_CHECKING
from fixtures import require_server_reachable
from framework.core.scoutboard import ScoutConfig

if TYPE_CHECKING:
    from framework.page import Page
    from pages.device_configuration_page import DeviceConfigurationPage

OU_NAME = "my_scout_change"          # == the test's own name
TARGET_VALUE = "..."

def test_my_scout_change(desktop: Page, device_settings: DeviceConfigurationPage, scout_config_change, target_device_name: str):
    """<short behavior statement>"""
    require_server_reachable()

    desktop.step("Configuring Scout <setting> and rebooting...")
    scout_config = ScoutConfig()
    scout_config.<section>.<field> = TARGET_VALUE
    scout_config_change(OU_NAME, scout_config, device_name=target_device_name, reboot=True, wait_online=True, reboot_on_teardown=False)

    # ... act via Page Object methods (each self-verifying, narrated with page.step()) ...
    # ... assert ground truth (X11 if AT-SPI has nothing) ...
```

---

## Step 9 -- Run it live, iterate

`python -m pytest tests\test_my_scout_change.py -v -s` (allow 2-4 minutes for a reboot-involving run).
Between iterations, re-check the device is left clean (`ou_exists(...)` False, device back `ONLINE` in its
original `GroupID`, no leftover apps) -- a failed/interrupted run can leave a throwaway OU behind if
`apply_scout_ou_config_change()`'s own failure-safety cleanup doesn't cover a new failure mode you
introduced; clean it up manually (`client.delete_ou(path=..., force_delete_ou_filter=True)`) before the
next attempt.

---

## Step 10 -- Ship it: branch + PR per touched repo

If you only changed `elux-automation-client`, one PR. If Step 5 also touched
`elux-automation-server`, **two** PRs, cross-linked in each description (client PR notes it depends on the
server PR; server PR notes the paired client PR):

```powershell
# in each touched repo:
git checkout -b feature/<short-name>
git add -A
git commit -F <message-file>          # use a FILE for multi-line messages -- a PowerShell
                                       # here-string passed directly to `git commit -m` gets
                                       # mis-split into multiple pathspec arguments
git push -u origin feature/<short-name>
gh pr create --title "..." --body-file <body-file> --base master --head feature/<short-name>
```

Report both PR URLs back to whoever asked for this.

---

## Pre-flight checklist

- [ ] elux-ui-explorer used for any new AT-SPI locators; elux-e2e-test-author conventions followed for
      the Page Object/test shape.
- [ ] `scout_config_change`/`target_device_name` used -- no hand-rolled OU creation, no hardcoded device name.
- [ ] OU name matches the test's own name, if that was requested.
- [ ] `reboot_on_teardown=False` unless the live running session genuinely needs resetting too.
- [ ] Any assertion AT-SPI can't cover uses an X11-level `Page` primitive (`active_window`/`window_exists`/
      `window_action`/`keyboard.press`) -- not a coordinate, not a skipped check.
- [ ] A genuinely missing primitive was added via the 4-file server pattern, deployed, and smoke-tested
      live BEFORE the test that depends on it was written.
- [ ] A broken/rejected Scout REST call was checked against the OpenAPI spec and reproduced on a
      throwaway OU before assuming a server-side gap and reaching for an internal-endpoint fallback.
- [ ] Post-reboot code waits for a REAL action to succeed (not just `/health`) before returning control.
- [ ] The test body itself is steps-only; every loop/retry lives in a Page Object or fixture.
- [ ] Ran live, confirmed the device is left exactly as found (no leftover OU/apps, back in its original
      group/config).
- [ ] A branch + PR opened in every touched repo, cross-linked if more than one.

## Anti-patterns to avoid

- Calling `ScoutBoardClient.apply_config()`/`move_device_to_ou()` directly from a test instead of
  `scout_config_change`.
- Hardcoding a Scout device name instead of resolving it via `target_device_name`/`find_device_by_ip()`.
- Rebooting twice when the test doesn't need the live session reset (skip `reboot_on_teardown`).
- Faking ground truth with a screen coordinate or a bare `time.sleep()` when a real X11 primitive
  (or, one layer down, a new server route) is the correct fix.
- Assuming a rejected Scout REST call means your payload is wrong, without first checking the OpenAPI
  spec and reproducing on a disposable OU.
- Reaching for an undocumented internal endpoint as a first resort instead of a last one, or leaving it
  un-gated so every caller has to know about the workaround.
- Trusting `/health`'s `atspi_connected: true` alone as "the UI is ready" right after a reboot.
- Leaving a throwaway Scout OU or test-created app behind because a NEW failure mode wasn't covered by
  the existing cleanup -- verify the device is clean before calling the job done.
