---
name: elux-ui-explorer
description: >-
    Explores the live eLux desktop UI (a screenshot plus the AT-SPI accessibility tree) over
    elux-automation-server's HTTP API to help write new Locator/Page Object/e2e_tests code in the
    elux-automation-client repo, without needing SSH/shell access to the eLux device. Use when
    asked to write a new eLux UI test, add a new Page Object or locator, inspect the ucdesktop (or
    ucscreensaver/loginmanager/xfreerdp) accessibility tree, or figure out how to click/fill/
    assert on a specific eLux UI element.
user-invocable: true
---

# eLux UI Explorer (screenshot + AT-SPI tree -> new test)

This skill teaches the screenshot+tree exploration loop used to write a NEW test against the
eLux desktop shell in **elux-automation-client**, driving a real device through
**elux-automation-server**'s HTTP API. Two repos are involved:

- `elux-automation-server` -- runs ON the eLux device, exposes `GET /x11/screenshot` (what the UI
  looks like) and `GET /atspi/tree` (what every element's exact role/name/state is).
- `elux-automation-client` -- where you actually write the `Locator`/Page Object/`e2e_tests/test_*.py`
  code, using the Page Object Model under `framework/pages/`.

**The core loop, every time you need to locate a new element:**
1. Confirm the server is reachable.
2. Screenshot the current screen to see what's visually there.
3. Query the AT-SPI tree (optionally filtered) for that screen to get exact role/name/state.
4. Turn one tree node into a locator path step.
5. Verify the locator actually resolves with `POST /atspi/find` BEFORE writing any Page Object code.
6. Write/extend the Page Object, then the test. Run it.

---

## Step 0 -- Confirm the server is reachable

The client repo's `.env` (copy from `.env.example`) or `ELUX_SERVER_URL` env var points at the
server, default `http://127.0.0.1:5000`. Check it:

```powershell
# PowerShell
curl.exe -s http://<device-ip>:5000/health
```
```bash
# curl (Linux/macOS/WSL)
curl -s http://<device-ip>:5000/health
```
Expect `{"ok":true,"result":{"status":"ok","atspi_connected":true,...}}`. If unreachable:
- The server may not be running -- see elux-automation-server's README ("Running"/"Deploying"):
  either `systemctl start elux-automation-server` (as root, if deployed as a service) or, for a
  quick dev iteration, SSH in and run `python3 app.py` directly from its deployed directory.
- If the device's HTTP port isn't directly reachable from where you're working, open an SSH
  tunnel: `ssh -i <key> -L 5000:localhost:5000 <user>@<device-ip>`, then use
  `http://127.0.0.1:5000` as the server URL.

Set it for the rest of this workflow:
```powershell
$env:ELUX_SERVER_URL = "http://<device-ip>:5000"
```

---

## Step 1 -- Screenshot: see what's on screen right now

From the **elux-automation-client** repo root:
```powershell
python framework\tools\inspect_ui.py --list-apps      # if you don't know the app name yet
python framework\tools\inspect_ui.py                   # screenshot + full tree of the default app
```
The plain screenshot-only call:
```powershell
curl.exe -s -o screen.png http://<device-ip>:5000/x11/screenshot
```
View `screen.png` (or the PNG `inspect_ui.py` saves under `e2e_tests/test-results/inspect/`) to
see the exact current state -- which window/panel is open, what's visible, what's not. **AT-SPI
only reports elements that are currently rendered** -- if the screen/panel you need isn't open
yet, drive the UI there first (e.g. `page.start_menu.open()` then `page.device_settings.open()`
in a short Python snippet, or real clicks via `/x11/mouse/click` on visible coordinates from this
screenshot) before dumping its tree.

---

## Step 2 -- AT-SPI tree: get exact role/name/state

**Preferred: `framework/tools/inspect_ui.py`** (screenshot + tree in one command, run from the
client repo root):
```powershell
python framework\tools\inspect_ui.py --filter "time zone"
python framework\tools\inspect_ui.py --app ucdesktop --filter "time zone" --save-json
python framework\tools\inspect_ui.py --app ucdesktop --depth 6 --no-screenshot   # full dump, tree only
```

**Or call the server route directly** (`GET /atspi/tree`, query params all optional):
```powershell
curl.exe -s "http://<device-ip>:5000/atspi/tree?app_name=ucdesktop&filter=time%20zone"
```
| param      | meaning                                                                              |
|------------|----------------------------------------------------------------------------------------|
| `app_name` | app to dump (`ucdesktop`, `ucscreensaver`, `loginmanager`, `xfreerdp`, ...). Omit to list registered apps instead. |
| `filter`   | case-insensitive substring against name/role/description -- **use this whenever you have any idea what you're looking for**; a full unfiltered dump of a complex screen can be hundreds of nodes. |
| `depth`    | max tree depth (default 20) -- lower it for a quick shallow look, raise it for deeply-nested WebKit content. |

Response shape (`result`):
```json
{
  "app_name": "ucdesktop",
  "filter": "time zone",
  "depth_limit": 20,
  "match_count": 1,
  "tree_text": "ucdesktop/0/1/2: [combo box] name='Time zone' center=(640, 220) ...",
  "nodes": [
    {
      "path": "0/1/2",
      "name": "Time zone",
      "role_name": "combo box",
      "role_alias": "combo_box",
      "description": null,
      "rect": {"x": 600, "y": 200, "width": 80, "height": 40},
      "center": {"x": 640, "y": 220},
      "child_count": 0,
      "states": ["enabled", "visible", "showing", "focusable"],
      "actions": [],
      "interfaces": ["Component", "Text"],
      "line": "[combo box] name='Time zone' center=(640, 220) ..."
    }
  ]
}
```
Without `app_name`, `result` is instead just `{"applications": ["ucdesktop", "ucscreensaver", ...]}`.

---

## Step 3 -- Turn a node into a locator path step

A locator path step is `{"by": "...", "value": "...", "role": "..."}` (role optional depending on
`by`). Map a tree node's fields like this:

| `by`            | when to use it                                                                    | value comes from        |
|-----------------|-------------------------------------------------------------------------------------|--------------------------|
| `name`          | element has a distinct accessible `name`, role doesn't matter                        | node `name`               |
| `role_name`     | element has a `name` AND you want to also pin down the role (recommended default)    | node `name` + `role_alias` |
| `role`          | only one element of this role exists in scope, no usable name                        | node `role_alias`         |
| `text_contains` | forgiving substring match across name/role/description (exploratory)                 | any distinctive substring |
| `content`       | element's accessible `name` is empty but it has visible Text-interface content (many WebKit headings/labels) | the exact visible text |
| `description`   | element has no usable name but a distinct `description`                              | node `description`        |
| `editable` / `editable_text` | a nameless WebKit `embedded` input field (no `name`, `role_name` is "embedded") | role="embedded", optionally the field's live text |

`role_alias` (when non-null) is EXACTLY the string this codebase's locators use for `role` --
copy it verbatim (see `elux-automation-server/server/atspi_utils.py`'s `ROLE_ALIASES` for the
full registered set: `push_button, toggle_button, menu, menu_item, label, text, password_text,
embedded, icon, frame, document_frame, dialog, panel, combo_box, list_item, slider, check_box`).
If `role_alias` is `null` for an element you need to target by role, it isn't registered yet --
either use `name`/`text_contains`/`content` instead, or add a new alias to `ROLE_ALIASES`
server-side (small, tightly-scoped server change) if targeting by role is really needed.

Example, from the `nodes[0]` above:
```json
{"by": "role_name", "value": "Time zone", "role": "combo_box"}
```

A **locator chain** (scoping a search to inside another element, e.g. a slider inside a specific
labelled group) is just an ORDERED LIST of these steps -- mirrors chained calls like
`page.get_by_name("Screen brightness").get_by_role("slider")` in existing Page Objects.

---

## Step 4 -- Verify BEFORE writing any code

Never write Page Object code against a guessed locator -- verify it resolves first:
```powershell
curl.exe -s -X POST http://<device-ip>:5000/atspi/find `
  -H "Content-Type: application/json" `
  -d '{"app_name": "ucdesktop", "path": [{"by": "role_name", "value": "Time zone", "role": "combo_box"}], "timeout": 3, "poll_interval": 0.2}'
```
Or, from Python (either repo has network access -- this uses the client's own thin HTTP client):
```python
import sys; sys.path.insert(0, ".")   # run from elux-automation-client repo root
from framework.core.server_client import ServerClient

client = ServerClient("http://<device-ip>:5000")
path = [{"by": "role_name", "value": "Time zone", "role": "combo_box"}]
print(client.atspi_find("ucdesktop", path, timeout=3, poll_interval=0.2))
```
A 404 `ElementNotFoundError` means the locator doesn't resolve yet -- go back to Step 2 (is the
right screen/panel actually open? try a broader `filter`, or re-check `role_alias`/`name` spelling
exactly as returned -- these are case-sensitive exact matches for `name`/`role_name`).

---

## Step 5 -- Write/extend the Page Object

Follow the existing pattern in `framework/pages/*.py` (e.g. `date_and_time_page.py`): a class
taking `page` in `__init__`, `@property` accessors returning `Locator`s (via
`self.page.get_by_role(...)`, `.get_by_name(...)`, `.get_by_text(...)`, `.get_by_content(...)`, or
the generic `self.page.locator(by, value, role=...)` escape hatch), and self-verifying action
methods that raise `framework.core.errors.WaitTimeoutError` on failure and `return self` for
chaining. Reuse `self.page.step("...")` for narration (also drives
`framework/tools/demo_set_timezone.py`-style human-watchable demos for free).

```python
@property
def timezone_field(self):
    return self.page.get_by_role("combo_box", name="Time zone")
```

---

## Step 6 -- Write the test and run it

```python
# e2e_tests/test_something.py
def test_something(page):
    page.device_settings.open()
    page.device_settings.date_and_time.open()
    page.device_settings.date_and_time.set_timezone("Europe / Sarajevo")
```
```
pytest -v e2e_tests/test_something.py
```
Iterate: if a step fails, re-run Steps 1-4 for that exact element (the screen state may differ
from what you inspected, or a WebKit-rendered subtree may need a moment to populate -- AT-SPI
trees for WebKit content populate lazily/asynchronously; re-querying `/atspi/tree` a moment later,
or letting the real locator's built-in retry/timeout handle it, is normal, not a bug).

---

## Quick reference: full example session

```powershell
$env:ELUX_SERVER_URL = "http://10.101.149.194:5000"
cd elux-automation-client

# 1. What apps are up?
python framework\tools\inspect_ui.py --list-apps

# 2. Open the screen you need first (small ad hoc script), THEN inspect it
python -c "from framework.page import Page; p = Page(); p.start_menu.open(); p.device_settings.open(); p.device_settings.date_and_time.open()"

# 3. Screenshot + filtered tree for the panel you just opened
python framework\tools\inspect_ui.py --filter "time zone" --save-json

# 4. Verify the locator resolves
python -c "from framework.core.server_client import ServerClient; c = ServerClient('http://10.101.149.194:5000'); print(c.atspi_find('ucdesktop', [{'by':'role_name','value':'Time zone','role':'combo_box'}], 3, 0.2))"

# 5. Write the Page Object property / test, then:
pytest -v e2e_tests/test_something.py
```

## Troubleshooting

- **`ServerUnreachableError`** -- server down/unreachable network-wise; see Step 0.
- **`ElementNotFoundError` (404) from `/atspi/find`** -- the path never resolved within `timeout`;
  re-check the screen is actually open (Step 1's screenshot), broaden or fix the `filter` (Step
  2), and double check `name`/`role_name` values are byte-for-byte exact (case-sensitive).
- **`/atspi/tree` 404 "application not found"** -- the error message includes the currently
  registered app list; the app may not be launched yet, or its name differs from what you
  expected (e.g. the lock screen is `ucscreensaver`, not `lockscreen`).
- **Huge tree_text/nodes output** -- always pass `filter` once you have any idea what you're
  looking for; reserve a full unfiltered dump for a first look at a totally unfamiliar screen,
  and consider lowering `depth`.
- **A nameless WebKit `embedded` input** -- these have no accessible `name`; use `role="embedded"`
  with `by="editable"` (any editable field) or `by="editable_text"` (editable field whose live
  text equals a value) instead of `name`/`role_name`.
