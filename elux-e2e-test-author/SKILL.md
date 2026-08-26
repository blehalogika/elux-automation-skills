---
name: elux-e2e-test-author
description: >-
    Writes a new end-to-end test (and any supporting Page Object) for the eLux desktop shell in the
    elux-automation-client repo, following the project's Playwright-flavored Page Object Model + pytest
    conventions: coordinate/resolution-independent AT-SPI role/name locators, teardown-first cleanup via
    page.defer_restore(), ground-truth/behaviour-level verification (not just the GUI value), ticket-linked
    docstrings, live step narration, and shared setup for reboot-heavy Scout tests. Use when asked to
    write, add, or extend an eLux UI/e2e test, a tests/test_*.py file, or a pages/*.py Page Object in
    elux-automation-client. Pair with the elux-ui-explorer skill, which finds the AT-SPI elements this
    skill then writes tests against.
user-invocable: true
---

# eLux e2e test author (Page Object Model + pytest conventions)

This skill teaches how to WRITE a new test against the eLux desktop shell (`ucdesktop`) in
**elux-automation-client**, following the conventions already established by the suite under
`tests/` and the Page Objects under `pages/`. It picks up where **elux-ui-explorer** leaves off:

- `elux-ui-explorer` -> *find* an element (screenshot + AT-SPI tree + verify a locator resolves).
- **this skill** -> *turn that element into a Page Object + a well-structured, self-cleaning,
  behaviour-verifying test.*

The project is a client/server split (see `elux-automation-client/README.md`): the client owns the
Page Object Model and the pytest suite and drives a real device over HTTP; **elux-automation-server**
runs on the device and is the only thing that touches AT-SPI/X11/pactl. You never call AT-SPI
directly from a test -- you go through `Page`/`Locator`.

---

## The five principles (non-negotiable -- every existing test follows them)

1. **Locate by AT-SPI role/name, never by coordinates.** No hard-coded pixel coordinates, colours,
   regions, or resolution assumptions. Use `page.get_by_role(...)` / `get_by_name(...)` /
   `get_by_content(...)`. Where an image must be inspected (e.g. an RDP session), measure *change*
   in a resolution-independent way (see `framework/core/image.py::Screenshot.differs_from`), not an
   absolute pixel/colour at a fixed point.
2. **Verify the ground truth / behaviour, not just the widget value.** Assert the real backend or a
   user-visible effect, e.g. the libinput driver state, the synced `terminal.ini`, a process that
   actually launched -- not merely that a checkbox looks ticked. "behaviour-level check, not just
   the value" recurs all over `tests/`.
3. **Teardown-first.** Snapshot the original value and register its restore with
   `desktop.defer_restore(...)` (or a Page Object's `restore_*_after_test()` helper) **before** you
   mutate anything. No `try/finally` in the test body -- the afterEach reset runs the restores LIFO.
4. **Let locators auto-wait; never `time.sleep()` a UI.** Locators re-resolve fresh on every action
   and `expect(...)` retries until it passes or times out. For non-Locator conditions use
   `page.wait_for(predicate, timeout=...)`.
5. **One coherent scenario per test, linked to its ticket, narrated with `page.step(...)`.** The
   module docstring names the ticket (e.g. `ELUXQ-138`), states what user-facing behaviour it
   proves, and -- crucially -- explains *why* any deviation from the literal ticket wording is
   equivalent.

---

## Where things live (current repo layout)

```
elux-automation-client/
  pages/            One Page Object per screen (start_menu_page.py, local_mouse_settings_page.py, ...).
  tests/            The pytest suite. Add a test = add a tests/test_*.py file here.
    conftest.py     Wraps fixtures/ factories as @pytest.fixture; installs the per-test step recorder.
  fixtures/         Plain factory functions (require_server_reachable, require_env, reset_desktop_state, ...).
  framework/        The Page / Locator / expect engine (page.py, locator.py, expect.py, config.py, core/).
    tools/inspect_ui.py   Screenshot + AT-SPI tree dump -- the elux-ui-explorer entry point.
  test-results/     Per-run failure screenshots + HTML report (gitignored).
  pytest.ini        testpaths = tests; live per-step logging (log_cli).
```

**Fixtures already available** (request by name in a test signature): `desktop`, `start_menu`,
`power_options`, `quick_access`, `device_settings`, `lock_screen`, `login_screen`, `rdp`,
`windows_rdp_session`, `notifications`, `local_mouse_settings`, `scout_board`, `scout_config_change`.
Every per-area fixture derives from the one `desktop` fixture, so they share a single desktop Page.

The `desktop` fixture is your afterEach: it calls `fixtures.reset_desktop_state()` before **and**
after the test (closing any leftover window/panel/popup) and runs your deferred restores.

---

## Step 1 -- Find the elements (use the elux-ui-explorer skill)

Before writing code, get the exact `role`/`name`/`state` of each element and verify a locator
resolves with `POST /atspi/find`. That whole loop is the **elux-ui-explorer** skill -- follow it,
then come back here. Quick reminder from the client repo root:

```powershell
python framework\tools\inspect_ui.py --filter "time zone"
```

---

## Step 2 -- Write / extend the Page Object

One class per screen under `pages/`, taking `page` in `__init__` (optionally extending
`pages.base_page.BasePage`). Expose `@property` accessors that return `Locator`s, and
self-verifying action methods that narrate with `self.page.step(...)` and `return self` for
chaining. Prefer `get_by_role(role, name=...)`; fall back to `get_by_name`, `get_by_content`
(exact live WebKit Text with an empty accessible name), or the generic `locator(by, value, role=...)`.

```python
# pages/date_and_time_page.py
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:                       # type-only import: no runtime import cycle
    from framework.page import Page


class DateAndTimePage:
    def __init__(self, page: Page):
        self.page = page

    @property
    def timezone_field(self):
        # role + exact accessible name is the idiomatic default (see elux-ui-explorer).
        return self.page.get_by_role("combo_box", name="Time zone")

    def set_timezone(self, name: str) -> "DateAndTimePage":
        self.page.step(f"Select time zone {name!r}")
        self.timezone_field.click()
        self.page.get_by_role("menu_item", name=name).click()
        return self
```

Known role aliases (from `server/atspi_utils.py`): `push_button, toggle_button, menu, menu_item,
label, text, password_text, embedded, icon, frame, document_frame, dialog, panel, combo_box,
list_item, slider, check_box`.

**Put the teardown helper on the Page Object**, snapshotting NOW and deferring the restore:

```python
    def restore_timezone_after_test(self):
        original = self.current_timezone()                 # snapshot while it's known
        self.page.defer_restore(lambda: self.set_timezone(original))
```

---

## Step 3 -- If AT-SPI can't drive it, go through a server route (ground-truth backend)

Some controls are not robustly AT-SPI-drivable (e.g. an unlabelled WebKit slider like the
double-click *speed*). Don't fake coordinates -- drive the **real backend the shell honours** through
a dedicated server route and its `ServerClient` method, exactly as `local_mouse_settings_page.py`
does via `/mouse/double_click_time` (`org.mate.peripherals-mouse`). Document *why* in the docstring.
This is the same "drive the ground-truth backend, not a missing AT-SPI signal" approach the volume
control uses via `pactl`. Adding a route is a small, tightly-scoped change in
**elux-automation-server** plus one thin `ServerClient` method in the client.

---

## Step 4 -- Write the test file

Mirror the shape of `tests/test_local_mouse_settings.py`. Annotated skeleton:

```python
"""
tests/test_screen_brightness.py -- ELUXQ-XYZ "<short ticket title>".

<One paragraph: the user-facing behaviour this proves. If the test deviates from
the literal ticket wording (e.g. no file manager exists on eLux, so a desktop
icon stands in for a folder), explain WHY the substitute is behaviourally
identical -- this project explicitly values that reasoning.>

Prerequisites:
  - elux-automation-server reachable at Config.server_url.
  - AD_USERNAME / AD_PASSWORD in a repo-root .env (only if the test signs in).

Run with:
    pytest tests/test_screen_brightness.py -v
"""
from __future__ import annotations

from typing import TYPE_CHECKING

from fixtures import require_env

if TYPE_CHECKING:
    from framework.page import Page
    from pages.quick_access_page import QuickAccessPage

#: Module-level constants get an explanatory comment (why THIS value).
TARGET_BRIGHTNESS = 20


def test_screen_brightness_applies_immediately(
    desktop: Page,
    quick_access: QuickAccessPage,
):
    """A brightness change in the Quick System Access panel takes effect
    immediately and is reflected in its backing value -- ELUXQ-XYZ."""
    # 0. Preconditions up front -- skip/fail fast before touching the UI.
    require_env("AD_PASSWORD")

    # 1. Teardown FIRST: snapshot + register the restore BEFORE mutating,
    #    so the afterEach reset puts the device back (no try/finally here).
    quick_access.restore_brightness_after_test()

    # 2. Act in small, narrated steps (each set_*/open() calls page.step()).
    quick_access.set_brightness(TARGET_BRIGHTNESS)

    # 3. Assert the BEHAVIOUR/backend, with a descriptive failure message.
    assert quick_access.brightness() == TARGET_BRIGHTNESS, (
        "brightness should apply immediately and read back as set"
    )
```

Conventions to keep:

- `from __future__ import annotations` at the top; type-only imports under `TYPE_CHECKING`.
- Preconditions (`require_env(...)`, `require_server_reachable(...)`) on the first lines.
- A tight docstring naming the ticket and the exact behaviour asserted.
- Numbered `# 1. ... # 2. ...` comments for the phases.
- Establish a clean baseline before acting (delete a leftover app, terminate a stray process).
- Assertions carry a message explaining the expected behaviour.

---

## Step 5 -- Clean up the right way (teardown-first, no try/finally)

Register every restore up front; the afterEach reset runs them best-effort in LIFO order:

```python
# Snapshot + defer -- BEFORE any mutation:
local_mouse_settings.restore_double_click_time_after_test()
apps.delete_after_test(APP_NAME)
desktop.defer_restore(lambda: desktop._server.process_terminate(LAUNCHED_PROCESS))
```

Why this and not `try/finally`: `reset_desktop_state()` already runs after every test, closes any
window/panel/popup left open, and executes your deferred restores -- so a test body stays a linear
"arrange -> act -> assert" with no cleanup noise, and cleanup still happens even if an assertion
raises. Restores are held per-Page and each test gets a fresh Page, so they never leak between
tests. Prefer a named element (e.g. toggling the SystemBar button) over clicking bare desktop
coordinates when dismissing something.

---

## Step 6 -- Assert behaviour / ground truth, not the widget

Push the assertion down to the real effect. Real examples in the suite:

- **Driver level:** after moving the Acceleration slider, assert the libinput pointer's Accel Speed
  actually changed (`accel_speed_at_driver()`); after toggling Left-handed, assert the device
  reports the swap (`left_handed_active_at_driver()`). Proves config -> mate-settings-daemon ->
  input driver, not just a gsettings write.
- **Config file:** for Scout-pushed settings, assert every field synced to `/setup/terminal.ini`
  (`TerminalIniPage.assert_keyboard_mouse_synced(...)`).
- **User-visible effect:** a normal-speed double-click on a desktop icon actually *launches* it
  (the target process appears) at a slow double-click speed, and does *not* at the fastest speed.
- **Live screen change:** for an RDP session, prove the remote screen *reacted* to a keyboard probe
  via `Screenshot.differs_from(...)` -- change detection, no fixed coordinate/colour.

---

## Step 7 -- Register a fixture (only if you added a new Page Object)

Expose the new screen on `Page` as a stateless `@property`, then add a thin fixture in
`tests/conftest.py`:

```python
# framework/page.py
@property
def date_and_time(self) -> "DateAndTimePage":
    from pages.date_and_time_page import DateAndTimePage
    return DateAndTimePage(self)
```
```python
# tests/conftest.py
@pytest.fixture
def date_and_time(desktop: Page) -> DateAndTimePage:
    """The Date & Time settings screen -- e.g. date_and_time.set_timezone(...)."""
    return desktop.date_and_time
```

(If it's a sub-screen reached only through another Page, you may skip the top-level fixture and
navigate via `desktop.device_settings.date_and_time` instead.)

---

## Step 8 -- Run it

```powershell
pip install -r requirements.txt          # first time on this control machine
pytest tests\test_screen_brightness.py -v
```

- Each `page.step("...")` streams to the terminal live as the test runs (pytest.ini's `log_cli`),
  numbered `STEP N: ...`.
- On failure, a screenshot is saved under `test-results/<run timestamp>/<test name>/failure.png`,
  and the HTML report (when `.test-samples/conftest.py` is present locally) lands in the same run
  folder.
- A test skips (not fails) if the server is unreachable, and fails with a copy-paste hint if a
  required env var / credential is missing.

---

## Reboot-heavy or otherwise expensive setup? Share it (module-scoped fixture)

Applying a Scout Board config and rebooting the device costs minutes. When several tests only need
"config applied + device rebooted, then validate", bundle ONE combined config into ONE OU, reboot
ONCE, and share it via a `scope="module"` fixture whose teardown (move device back, delete OU) also
runs once -- see `tests/test_scout_keyboard_language_mouse_settings.py`'s `shared_setup`. Use
`reboot_on_teardown=False` to keep the whole module to a single restart, and
`require_server_reachable(wait=...)` to poll while a delayed self-reboot settles.

---

## Pre-flight checklist (definition of done)

- [ ] Elements located by role/name and verified to resolve (elux-ui-explorer), zero coordinates/colours.
- [ ] Actions live on a `pages/*.py` Page Object, narrated with `page.step(...)`, returning `self`.
- [ ] Preconditions (`require_env` / `require_server_reachable`) on the first lines.
- [ ] Every mutation has a snapshot + `defer_restore` / `restore_*_after_test` registered BEFORE it.
- [ ] No `try/finally` cleanup and no `time.sleep()` waiting on the UI.
- [ ] Assertions check the backend/behaviour, each with a descriptive message.
- [ ] Module docstring names the ticket, the behaviour, and justifies any deviation from it.
- [ ] New Page Object also exposed as a `Page` property + a `conftest.py` fixture.
- [ ] `pytest tests/test_...py -v` passes and leaves the device exactly as it was found.

---

## Anti-patterns to avoid

- Hard-coded pixel coordinates, colours, screen regions, or resolution assumptions.
- `time.sleep()` to "wait for" the UI -- use auto-waiting locators, `expect(...)`, or `page.wait_for(...)`.
- `try/finally` (or leaving state changed) instead of teardown-first `defer_restore`.
- Asserting only the on-screen widget value while ignoring whether the setting actually took effect.
- One sprawling test covering several unrelated behaviours -- split them (share setup if it's slow).
- Reaching past `Page`/`Locator` into raw AT-SPI/X11 from a test -- add a Page Object or a server route.
- Synthetic window close (`window_action("close", ...)`) for the Device configuration window -- it
  crashes the shared `ucdesktop` process; use `page.close_window_titlebar(...)` (a real title-bar click).
