---
name: elux-e2e-builder
description: >-
    End-to-end orchestrator that turns ANY request to add/verify eLux desktop-shell behavior into a
    shipped, working test -- not just draft code that looks right. Explores the live UI
    (elux-ui-explorer), writes a clean POM-conventions test (elux-e2e-test-author), RUNS IT LIVE against
    the real device, and iterates: diagnosing failures from actual server/device state, extending
    elux-automation-client and/or elux-automation-server with a new Page Object method, fixture, or server
    route ONLY when genuinely missing, and re-running -- until the test passes and the device is left
    clean, then ships a branch + PR per touched repo. Use whenever asked to write, add, fix, or verify an
    eLux e2e test or feature end-to-end (this includes, but is not limited to, Scout Board config-change
    tests, reboot-dependent settings, or anything needing a brand-new client/server capability).
user-invocable: true
---

# eLux e2e builder (explore -> write -> run live -> fix -> ship)

This is the outer loop that ties **elux-ui-explorer** and **elux-e2e-test-author** together into one
repeatable process for turning ANY request into a real, passing, shipped test. The defining trait of this
skill is: **never declare a test done until it has actually run against the live device and passed, and
the device has been verified clean afterward.** Everything below applies regardless of what the specific
test is about -- a Scout Board config change is just one possible instance of Step 1's "does this touch
backend state" question, handled the same way as any other case.

## The loop, at a glance

1. **Understand** the request -> restate it as ONE test's behavior + how you'll verify it (ground truth).
2. **Explore** (elux-ui-explorer) -> find every element you'll need, verify each locator resolves.
3. **Write** (elux-e2e-test-author) -> Page Object methods + a short, steps-only test.
4. **Run it live.** Read the ACTUAL failure (not a guess) -- exception type, live AT-SPI tree/screenshot,
   device/server state.
5. **Fix the ROOT CAUSE at the right layer** (see the table in Step 5) -- never patch around a failure in
   the test body itself.
6. **Re-run.** Repeat 4-5 until green.
7. **Verify** the device/environment is left exactly as found.
8. **Ship**: branch + PR per touched repo.

---

## Step 1 -- Turn the request into one behavior + its ground truth

Before writing anything, answer:

- What's the ONE user-facing behavior this test proves? Not "click these buttons" -- e.g. "the switch
  applications hotkey actually changes focus", "brightness set in Quick Access persists after reboot",
  "an added local application actually launches".
- What's the **ground truth** that this really happened, independent of the widget you clicked? See the
  table below for common cases -- always prefer the real backend/effect over the on-screen value.
- Does this touch anything beyond plain AT-SPI UI -- a Scout Board config value, a config file, a running
  process, a real reboot? If so, flag it now (Step 3b covers the reusable Scout workflow) rather than
  discovering it mid-test.

| If the behavior involves... | Assert this ground truth |
|---|---|
| A UI toggle/value | The underlying driver/config/process (e.g. libinput's accel speed, `/setup/terminal.ini`, a launched process) -- not just the widget |
| A Scout Board config push | Read the value back via Scout's own GET, AND its real effect on the device |
| A global hotkey / WM-level action | X11 (`Page.active_window()`/`window_action()`), not AT-SPI |
| A non-AT-SPI-aware window (plain xterm, etc.) | `Page.window_exists()`/`window_action()`/`active_window()` |
| A reboot-dependent setting | Apply with `reboot=True`; verify only AFTER the device is confirmed genuinely ready (Step 5's readiness row) |

---

## Step 2 -- Explore (elux-ui-explorer, unchanged)

Screenshot + `/atspi/tree` (filtered), verify every locator with `/atspi/find` BEFORE writing any Page
Object code. If the screen you need isn't open yet, drive there first with a throwaway script, then
inspect. Re-inspect (don't assume) whenever a screen's state might have changed -- e.g. after a reboot.

## Step 3 -- Write (elux-e2e-test-author, unchanged)

Role/name locators, Page Object methods that self-verify and `return self`, teardown-first via
`defer_restore()`/`delete_after_test()`, a tight docstring naming the ticket (or stating there is none)
and the exact behavior, numbered phase comments, a **steps-only test body** -- every loop/retry lives in a
Page Object or fixture, never inline in the test.

**3b. If the request touches Scout Board config**, reuse the already-built workflow instead of hand-rolling
OU/device juggling:

```python
def test_x(desktop, device_settings, scout_config_change, target_device_name):
    scout_config = ScoutConfig()
    scout_config.<section>.<field> = <value>
    scout_config_change(
        "<name-matching-the-test>", scout_config, device_name=target_device_name,
        reboot=True, wait_online=True, reboot_on_teardown=False,   # see the readiness row below
    )
```

- `target_device_name` resolves the Scout-registered device name from `ELUX_SERVER_URL`'s own host
  (`find_device_by_ip()`) -- never hardcode a device name.
- `scout_config_change` creates a throwaway OU (name it after the test if asked to), applies the config,
  moves the device in, and auto-tears-down (move back + delete OU) regardless of outcome.
- `reboot_on_teardown=False` unless the test specifically needs the device's *live session* reset, not
  just Scout's own bookkeeping restored -- this alone roughly halves runtime on a reboot-heavy test.

## Step 4 -- Run it live, read the REAL failure

```powershell
python -m pytest tests\test_x.py -v -s   # from the client repo root; allow minutes if reboot-involving
```

On failure, don't guess -- look:
- The exact exception **type** (`ElementNotFoundError` vs `WaitTimeoutError` vs `X11ToolError` vs a plain
  HTTP error) tells you which layer broke.
- `python framework\tools\inspect_ui.py --app <app>` for what's ACTUALLY on screen right now.
- Device/Scout state directly (`get_device_status()`, `ou_exists()`, `/health`) if a config/reboot step
  failed.

## Step 5 -- Fix at the right layer (never patch the test body)

| Symptom | Fix here |
|---|---|
| A locator doesn't resolve | Back to Step 2 -- re-inspect; the screen state may differ from what you assumed |
| An action needs verifying/retrying | The Page Object method itself (self-verify + `raise WaitTimeoutError`) |
| A cleanup/restore never runs on failure | The fixture's teardown logic -- register teardown BEFORE the risky steps, wrap them in `try/except` calling it (a real bug found + fixed this way in `apply_scout_ou_config_change()`) |
| AT-SPI has no signal for something | One layer down: `Page.window_exists()`/`window_action()`/`active_window()`/`keyboard.press()` (X11, already available) |
| Neither AT-SPI nor an existing X11 primitive covers it | Extend `elux-automation-server` -- the 4-file pattern: `server/x11.py` -> `server/routes/x11_routes.py` -> `framework/core/server_client.py` -> `framework/page.py`. Deploy (`deploy\transfer_to_box.ps1`) and smoke-test the new route directly BEFORE the test depends on it |
| A documented REST/API call is rejected | Check its spec first (don't assume your payload is wrong); reproduce on a disposable/throwaway resource; only as a last resort fall back to an internal/undocumented endpoint (capture a real request, verify empirically which fields are actually validated vs. merely required-present), and gate it transparently behind the existing abstraction so callers never see the workaround |
| A step that "should just work" flakes right after a reboot | The environment isn't as ready as a health check claims -- wait for (and retry) the REAL first action, not just connectivity (`fixtures._wait_for_desktop_ui_ready()` is the template: click, verify the popup opened, close it again, retry on failure) |
| Two independent teardowns race each other (e.g. a device reboots while other cleanup is still running) | Diagnose the actual interleaving from the log first, then reorder/guard the fixtures or make the racing step idempotent/tolerant |

Whatever layer you fix, keep the change **minimal and scoped** to what's actually needed, matching the
style of its neighbors -- not a speculative rewrite.

## Step 6 -- Re-run, then verify clean

Repeat Steps 4-5 until green. Then confirm the environment is EXACTLY as found: no leftover OU/app/process,
device back online in its original state. A test isn't done until this is true, not just until pytest
prints "passed" once.

## Step 7 -- Ship

Branch + PR per touched repo (client always; server only if you extended it). Cross-link PRs in their
descriptions when more than one repo was touched.

```powershell
git checkout -b feature/<short-name>
git add -A
git commit -F <message-file>     # a FILE, not a PowerShell here-string piped to -m (gets mis-split
                                  # into multiple pathspec arguments)
git push -u origin feature/<short-name>
gh pr create --title "..." --body-file <body-file> --base master --head feature/<short-name>
```

Report every PR URL back to whoever asked for this.

---

## Pre-flight checklist

- [ ] The test's ONE behavior + its ground truth were both decided BEFORE writing code.
- [ ] Every locator came from elux-ui-explorer's find-then-verify loop -- nothing guessed.
- [ ] The test body is steps-only; every retry/verification loop lives in a Page Object or fixture.
- [ ] It was actually RUN against the live device -- not just reviewed -- and passed.
- [ ] Any fix went to the correct layer (Step 5's table), stayed minimal, and matched existing style.
- [ ] Any new server/client primitive was deployed and smoke-tested live before the test depended on it.
- [ ] The device/environment was confirmed clean afterward.
- [ ] A branch + PR was opened per touched repo.

## Anti-patterns to avoid

- Writing/reviewing code and calling it done without ever running it live.
- Patching a failure in the test body (an inline retry loop, a `time.sleep()`, a swallowed exception)
  instead of fixing the Page Object/fixture/server layer that actually broke.
- Guessing at a fix without reading the actual exception type / live screenshot / device state first.
- Reaching for an undocumented/internal API as a first resort instead of a last one.
- Declaring victory after the FIRST green run without checking the environment was left clean.
- Skipping the shipping step -- a test that only exists on a local branch isn't finished.
