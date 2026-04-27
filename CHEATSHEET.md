# Playwright Python Cheat Sheet

> `pip install playwright && playwright install` · [docs](https://playwright.dev/python) · Python ≥ 3.8

For async add `await` before every call and use `async_playwright` / `async with`.

---

## Contents

| | |
|--|--|
| [Setup](#setup) | [Navigation & Waiting](#navigation--waiting) |
| [Locators](#locators) | [Network](#network) |
| [Actions](#actions) | [Screenshots & Tracing](#screenshots--tracing) |
| [Assertions](#assertions) | [Config & pytest](#config--pytest) |
| [Quick Test Example](#quick-test-example) | [CLI](#cli) |
| [Page Object Model](#page-object-model) | [Key Patterns](#key-patterns) |
| [Best Practices](#best-practices) | |

---

## Setup

```python
# sync script
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False, slow_mo=50)
    context = browser.new_context(
        storage_state="auth.json",   # reuse saved session
        viewport={"width": 1280, "height": 720},
        locale="en-US",
        timezone_id="Europe/Prague",
    )
    page = context.new_page()
    page.goto("https://example.com")
    browser.close()
```

```python
# async script
import asyncio
from playwright.async_api import async_playwright

async def main():
    async with async_playwright() as p:
        browser = await p.chromium.launch()
        page = await browser.new_page()
        await page.goto("https://example.com")

asyncio.run(main())
```

```python
# pytest — conftest.py
@pytest.fixture(scope="session")
def browser_context_args(browser_context_args):
    return {**browser_context_args, "storage_state": "auth.json"}
```

---

## Locators

| Priority | Use |
|----------|-----|
| `get_by_test_id("id")` | **Best** — add `data-testid` to your elements |
| `get_by_role("button", name="OK")` | ARIA role + visible name |
| `get_by_label("Email")` | Form inputs |
| `get_by_text("Sign in")` | Visible text |
| `get_by_placeholder("Search…")` | Input placeholder |
| `locator("css")` / `locator("xpath=…")` | Last resort |

```python
# chaining & filtering
page.get_by_role("listitem").filter(has_text="Item 2").get_by_role("button").click()
page.get_by_role("listitem").filter(has=page.get_by_role("heading", name="Item 2"))
page.get_by_role("button").and_(page.get_by_title("Delete"))

# lists
locator.first / locator.last / locator.nth(0)
locator.count()
for item in locator.all():
    item.click()

# iframe
frame = page.frame_locator("#iframe-id")
frame.get_by_role("button").click()

# OR — handle optional dialog
expect(btn.or_(dialog).first).to_be_visible()
if dialog.is_visible():
    dialog_close.click()

# custom test-id attribute (conftest.py)
playwright.selectors.set_test_id_attribute("data-cy")
```

---

## Actions

```python
locator.click()
locator.click(button="right")
locator.click(modifiers=["Shift"])
locator.dblclick()
locator.hover()
locator.tap()                          # mobile

locator.fill("text")                   # sets value directly
locator.press_sequentially("text", delay=50)  # simulates keystrokes
locator.press("Enter")
locator.press("Control+a")
locator.clear()

locator.check() / locator.uncheck()
locator.select_option("value")
locator.select_option(label="Blue")
locator.select_option(["a", "b"])      # multi-select
locator.set_input_files("file.pdf")

locator.scroll_into_view_if_needed()
page.mouse.wheel(0, 500)              # scroll down 500px

# read
locator.text_content()
locator.input_value()
locator.get_attribute("href")
locator.is_visible() / is_enabled() / is_checked()

# JS
page.evaluate("document.title")
page.evaluate("([a,b]) => a+b", [1, 2])
locator.evaluate_all("els => els.map(e => e.textContent)")

# dialogs
page.once("dialog", lambda d: d.accept())   # must register BEFORE action
```

---

## Assertions

> `expect()` auto-retries until timeout — always prefer over manual `is_visible()` checks.

```python
from playwright.sync_api import expect

# element state
expect(locator).to_be_visible()
expect(locator).to_be_hidden()
expect(locator).to_be_enabled() / to_be_disabled()
expect(locator).to_be_checked()
expect(locator).to_be_focused()
expect(locator).to_be_in_viewport()
expect(locator).to_be_attached() / not_.to_be_attached()

# content
expect(locator).to_have_text("exact")
expect(locator).to_have_text(re.compile(r"partial"))
expect(locator).to_contain_text("sub")
expect(locator).to_have_value("input value")
expect(locator).to_have_count(3)
expect(locator).to_have_attribute("href", "/path")
expect(locator).to_have_class("active")
expect(locator).to_have_css("color", "rgb(0,0,0)")

# page
expect(page).to_have_url("/dashboard")
expect(page).to_have_url(re.compile(r"/dashboard"))
expect(page).to_have_title("My App")

# response
expect(response).to_be_ok()

# modifiers
expect(locator).not_.to_be_visible()
expect(locator).to_be_visible(timeout=10_000)
expect.soft(locator).to_have_text("x")    # non-fatal
expect.poll(lambda: page.evaluate("window.done"), timeout=5_000).to_be_truthy()

# screenshots
expect(page).to_have_screenshot("home.png")
expect(locator).to_have_screenshot("widget.png")
# pytest --update-snapshots  to refresh
```

---

## Navigation & Waiting

```python
page.goto("https://example.com")
page.goto("/path")          # relative when base_url set
page.goto("/slow", wait_until="networkidle")  # domcontentloaded | load | networkidle | commit
page.reload()
page.go_back() / page.go_forward()

page.wait_for_url("**/dashboard")
page.wait_for_load_state("networkidle")

locator.wait_for(state="visible")   # visible | hidden | attached | detached

# wait for request/response triggered by an action
with page.expect_response("**/api/**") as r:
    page.get_by_text("Load").click()
resp = r.value
body = resp.json()

# download
with page.expect_download() as dl:
    page.get_by_text("Export").click()
dl.value.save_as("/tmp/file.csv")

# file upload dialog
with page.expect_file_chooser() as fc:
    page.get_by_text("Upload").click()
fc.value.set_files("report.pdf")

# popup / new tab
with page.expect_popup() as popup_info:
    page.get_by_text("Open").click()
popup = popup_info.value
popup.wait_for_load_state()
```

---

## Network

```python
import json

# mock response
page.route("**/api/items", lambda r: r.fulfill(
    status=200,
    content_type="application/json",
    body=json.dumps([{"id": 1}]),
))

# abort (block trackers, images, etc.)
page.route("**/*.{png,jpg}", lambda r: r.abort())

# modify real response
def patch(route):
    resp = route.fetch()
    data = resp.json()
    data["flag"] = True
    route.fulfill(response=resp, json=data)
page.route("**/api/flags", patch)

# direct API calls (shares cookies with page)
res = page.request.get("/api/users")
expect(res).to_be_ok()
data = res.json()

res = page.request.post("/api/items", data={"name": "x"})

# HAR replay
context.route_from_har("requests.har", not_found="fallback")

# listen to all requests
page.on("request",  lambda req:  print(req.method, req.url))
page.on("response", lambda resp: print(resp.status, resp.url))
```

---

## Screenshots & Tracing

```python
page.screenshot(path="shot.png")
page.screenshot(path="full.png", full_page=True)
page.screenshot(path="photo.jpg", type="jpeg", quality=80)
page.screenshot(clip={"x": 0, "y": 0, "width": 400, "height": 300})
page.screenshot(mask=[page.get_by_label("Card")], animations="disabled")

locator.screenshot(path="element.png")

# tracing
context.tracing.start(screenshots=True, snapshots=True, sources=True)
# ... test ...
context.tracing.stop(path="trace.zip")
# playwright show-trace trace.zip

# video
context = browser.new_context(record_video_dir="videos/")
# must close context to flush: context.close()
```

---

## Config & pytest

```ini
# pytest.ini
[pytest]
addopts = --browser chromium --headed
base_url = http://localhost:3000
tracing = retain-on-failure
screenshot = only-on-failure
video = retain-on-failure
```

```python
# conftest.py
import pytest, os
from playwright.sync_api import Page, expect

@pytest.fixture(autouse=True)
def _expect_timeout():
    expect.set_options(timeout=10_000)

# login once, reuse across all tests
@pytest.fixture(scope="session")
def auth_state(browser):
    ctx = browser.new_context()
    page = ctx.new_page()
    page.goto("/login")
    page.get_by_label("Email").fill(os.getenv("TEST_USER"))
    page.get_by_label("Password").fill(os.getenv("TEST_PASS"))
    page.get_by_role("button", name="Sign in").click()
    ctx.storage_state(path="auth.json")
    ctx.close()

@pytest.fixture
def auth_page(browser, auth_state) -> Page:
    ctx = browser.new_context(storage_state="auth.json")
    page = ctx.new_page()
    yield page
    ctx.close()

# device emulation
@pytest.fixture
def mobile_context(browser):
    iphone = playwright.devices["iPhone 14"]
    return browser.new_context(**iphone)
```

```python
# marks
@pytest.mark.skip(reason="WIP")
@pytest.mark.xfail
@pytest.mark.only_browser("chromium")
@pytest.mark.parametrize("url", ["/home", "/about"])
@pytest.mark.flaky(reruns=3, reruns_delay=2)   # pip install pytest-rerunfailures
```

---

## CLI

```bash
pytest                              # run all
pytest -k "login"                   # filter by name
pytest --headed                     # show browser
pytest --slowmo 300
pytest -x                           # stop on first fail
pytest -n 4                         # parallel (pip install pytest-xdist)
pytest --tracing=on
pytest --screenshot=on
pytest --update-snapshots

playwright codegen example.com
playwright codegen --device="iPhone 14" example.com
playwright codegen --save-storage=auth.json example.com

playwright show-trace trace.zip
playwright install --with-deps      # CI: install browsers + OS deps

PWDEBUG=1 pytest                    # opens Playwright Inspector
DEBUG=pw:api pytest                 # verbose API log
```

---

## Page Object Model

```python
from playwright.sync_api import Page, expect

class LoginPage:
    def __init__(self, page: Page):
        self.page     = page
        self.email    = page.get_by_test_id("email-input")
        self.password = page.get_by_test_id("password-input")
        self.submit   = page.get_by_role("button", name="Sign in")
        self.error    = page.get_by_test_id("login-error")

    def goto(self):          self.page.goto("/login")
    def login(self, e, p):   self.email.fill(e); self.password.fill(p); self.submit.click()
    def expect_error(self, text): expect(self.error).to_contain_text(text)

# test
def test_login(page):
    lp = LoginPage(page)
    lp.goto()
    lp.login("a@b.com", "pass")
    expect(page).to_have_url("/dashboard")
```

---

## Key Patterns

```python
# setup via API, assert in UI  (faster than UI setup)
resp = page.request.post("/api/orders", data={"item": "A"})
page.goto(f"/orders/{resp.json()['id']}")
expect(page.get_by_test_id("status")).to_be_visible()

# wait for spinner to disappear
page.locator(".spinner").wait_for(state="hidden")

# multi-user (two browsers, two sessions)
ctx_admin = browser.new_context(storage_state="admin.json")
ctx_user  = browser.new_context(storage_state="user.json")

# soft assertions — all run even on failure
expect.soft(page.get_by_test_id("title")).to_have_text("Dashboard")
expect.soft(page.get_by_test_id("menu")).to_be_visible()
expect(page).to_have_url("/dashboard")   # hard flush at end

# console / JS error capture
errors = []
page.on("pageerror", lambda e: errors.append(e))
# ... test ...
assert errors == [], str(errors[0])

# clock control
page.clock.set_fixed_time("2024-01-01T10:00")
page.clock.fast_forward("01:00")   # advance 1 hour

# cookies & storage
context.add_cookies([{"name": "s", "value": "abc", "domain": "x.com", "path": "/"}])
context.storage_state(path="state.json")
page.evaluate("localStorage.setItem('k','v')")

# geolocation
context = browser.new_context(
    geolocation={"latitude": 50.08, "longitude": 14.43},
    permissions=["geolocation"],
)
```

---

## Quick Test Example

A realistic test — login via saved state, set up data via API, assert in UI, clean up via API.

```python
# conftest.py
import pytest, os
from playwright.sync_api import Page, expect
from dotenv import load_dotenv

load_dotenv()

@pytest.fixture(autouse=True)
def _timeout(): expect.set_options(timeout=10_000)

@pytest.fixture(scope="session")
def auth_state(browser):
    ctx = browser.new_context()
    p   = ctx.new_page()
    p.goto("/login")
    p.get_by_label("Email").fill(os.getenv("TEST_USER"))
    p.get_by_label("Password").fill(os.getenv("TEST_PASS"))
    p.get_by_role("button", name="Sign in").click()
    ctx.storage_state(path="auth.json")
    ctx.close()

@pytest.fixture
def page(browser, auth_state) -> Page:
    ctx = browser.new_context(storage_state="auth.json")
    pg  = ctx.new_page()
    yield pg
    ctx.close()
```

```python
# tests/test_items.py
import pytest
from playwright.sync_api import Page, expect

def test_item_appears_in_list(page: Page):
    # Arrange — create via API (no UI clicks needed)
    resp = page.request.post("/api/items", data={"name": "Widget"})
    expect(resp).to_be_ok()
    item_id = resp.json()["id"]

    # Act
    page.goto("/items")

    # Assert
    expect(page.get_by_test_id(f"item-{item_id}")).to_be_visible()
    expect(page.get_by_test_id(f"item-{item_id}")).to_contain_text("Widget")

    # Teardown — clean up via API
    page.request.delete(f"/api/items/{item_id}")


def test_delete_item(page: Page):
    item_id = page.request.post("/api/items", data={"name": "ToDelete"}).json()["id"]
    page.goto("/items")

    page.once("dialog", lambda d: d.accept())                   # confirm dialog
    page.get_by_test_id(f"item-{item_id}").get_by_role("button", name="Delete").click()

    expect(page.get_by_test_id(f"item-{item_id}")).not_.to_be_attached()


def test_error_on_invalid_form(page: Page):
    page.goto("/items/new")
    page.get_by_role("button", name="Save").click()             # submit empty
    expect(page.get_by_test_id("field-error")).to_be_visible()
    expect(page).to_have_url("/items/new")                      # stayed on page


@pytest.mark.parametrize("name", ["", "x" * 300])
def test_validation(page: Page, name: str):
    page.goto("/items/new")
    page.get_by_label("Name").fill(name)
    page.get_by_role("button", name="Save").click()
    expect(page.get_by_test_id("field-error")).to_be_visible()


def test_network_error_shows_banner(page: Page):
    page.route("**/api/items", lambda r: r.fulfill(status=500))
    page.goto("/items")
    expect(page.get_by_test_id("error-banner")).to_be_visible()
```

---

## Best Practices

| ✅ Do | ❌ Don't |
|-------|---------|
| `get_by_test_id()`, `get_by_role()` | CSS classes, XPath |
| `expect(loc).to_be_visible()` | `time.sleep()` |
| Setup/teardown via API | UI clicks for every test |
| `storage_state` auth (session scope) | Login via UI per test |
| `expect.soft()` for independent checks | One giant assertion |
| `pytest -n 4` / `asyncio.gather()` | Shared browser across threads |
| `PWDEBUG=1`, `page.pause()`, traces | `print()` debugging |
