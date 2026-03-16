# Switch-to-Playwright-for-integration-testing-proposal-for-Google-Summer-of-Code-2026
## 1. Motivation
Django's integration tests currently lean on ``SeleniumTestCase``. Though it has served well in the past, it's plagued by sluggish run times, tests that sometimes fail because of implicit waits, and the need for fragile, browser-specific hacks to keep up with current testing demands. ``Playwright``, on the other hand, brings native auto-waiting, the ability to run tests in parallel, modern browser context isolation, and consistent APIs across different browsers.
To streamline the process of maintaining and writing end-to-end tests for the core team, I suggest a complete shift from Selenium to Playwright for Django's integration testing suite, ultimately leading to the full retirement of SeleniumTestCase.

### 1.1 Links
- [Ticket #13: Use Playwright for integration testing](https://github.com/django/new-features/issues/13)
- [My PlaywrightTestCase Proof of Concept (Async & Axe Integration)](https://github.com/VIZZARD-X/django/tree/gsoc-playwright-axe-poc/tests/admin_playwright)

## 2. Proposal
### 2.1. Thread-Safe PlaywrightTestCase
One of the biggest challenges that developers face when trying to use Playwright is that Playwright uses an asyncio event loop while Django uses a synchronous database flush operation during ``_fixture_teardown``, causing ``SynchronousOnlyOperation`` exceptions.
Instead of applying a global setting to disable thread protection or overriding the teardown methods by adding a lot of complex conditionals, we will properly scope the ``DJANGO_ALLOW_ASYNC_UNSAFE`` flag at the base class level using unittest’s more modern mechanism.
```python
class PlaywrightTestCase(LiveServerTestCase):
    @classmethod
    def setUpClass(cls):
        cls._old_async_unsafe = os.environ.get("DJANGO_ALLOW_ASYNC_UNSAFE")
        os.environ["DJANGO_ALLOW_ASYNC_UNSAFE"] = "true"
        cls.addClassCleanup(cls._restore_async_unsafe)
        super().setUpClass()
        cls.addClassCleanup(cls.playwright.stop)
```
### 2.2. Native Accessibility (Axe) Integration
Testing the admin panel for accessibility is currently fragmented. I will natively integrate ``axe-playwright-python`` into the base class.
```python
def assertNoAccessibilityViolations(self, axe_options=None):
    results = Axe().run(self.page, options=axe_options)
    self.assertEqual(
        results.violations_count, 0,
        f"Accessibility violations found:\n{results.generate_report()}"
    )
```

### 2.3. Test API Migration
When migrating the existing ~800 test methods, you will need to re-write them so they can run under Playwright. The main difference is that Selenium’s explicit ``wait_for`` helpers and ``find_elements`` call will be replaced with Playwright's built-in auto-waiting locators and ``expect`` assertions.
The number of lines of code (boilerplate) and the number of imports in the test files will be greatly reduced.

Before
```python
# Requires importing By, and relying on custom explicit waits
from selenium.webdriver.common.by import By

self.selenium.get(self.live_server_url + "/admin/")
self.wait_for_text("#content h1", "Django administration")

username_input = self.selenium.find_element(By.NAME, "username")
username_input.send_keys("admin")
```
After
```python
# Auto-waiting is native; no explicit wait helpers or By imports needed
from playwright.sync_api import expect

self.page.goto(self.live_server_url + "/admin/")
expect(self.page.locator("#content h1")).to_have_text("Django administration")

self.page.get_by_label("Username:").fill("admin")
```
### 2.4. Eradicating Selenium Technical Debt
An examination of ``django/test/selenium.py`` and ``django/contrib/admin/tests.py`` exposes multiple instances where Selenium necessitates fragile workarounds. This transition will remove these makeshift solutions.

#### 2.4.1 Cross-Browser Emulation: 
Emulating high-contrast and dark mode across multiple browsers requires using pure CDP commands ``(executeCdpCommand("Emulation.setEmulatedMedia", ...))`` - which limits testing capabilities to Chrome only. Playwright supports this functionality via native API calls for all supported browsers when calling ``page.emulate_media()``.
#### 2.4.2 Viewport Manipulation Hacks:
In order to force a DOM resize, the ``AdminSeleniumTestCase.trigger_resize()`` method modifies the width of the window by an increment of 1 pixel, first increasing the size of the window by 1 pixel and then decreasing it ``(self.selenium.set_window_size(width + 1, height))``. With Playwright's predictable rendering characteristics, there is no longer any need to implement such manual reflows.
#### 2.4.3 CSP Log Checking: 
Content Security Policy violations are currently being checked during the ``tearDown`` process using ``get_browser_logs``. However, on non-Chrome browsers, there is an empty list returned by this method because it only works with the ``goog:loggingPrefs`` capability. Instead of requiring that, Playwright has a unified method across all supported browsers for checking CSP violations using ``page.on('console')``.
#### 2.4.4 Brittle Navigation Waits:
``AdminSeleniumTestCase.wait_page_loaded()`` requires a ``try...except WebDriverException`` block specifically to handle a bug in Chrome driver 113+ regarding stale element references. Playwright’s native ``page.wait_for_load_state()`` completely deletes the need for this boilerplate.
#### 2.5. Test Runner & CI/CD
The DiscoverRunner will be extended with a ``--playwright flag``. A metaclass (mirroring the current ``SeleniumTestCaseBase``) will dynamically generate test variants for ``chromium``, ``firefox``, and ``webkit``. To prevent CI bloat, .github/workflows/tests.yml will utilize ``actions/cache`` keyed to the Playwright version.

### 3. Scope & Detailed Timeline
There are roughly ~800 Selenium test methods across Django. I propose that the complete migration, and the subsequent removal of Selenium, can be executed within the 175hr (Medium) scope.
(Note: The foundational research, PlaywrightTestCase prototype, and async boundary solutions have already been completed during the proposal phase).

Community Bonding Period (May)
- Finalize the ``PlaywrightTestCase`` architecture with mentors based on PoC feedback.
- Define the exact Axe ruleset mappings for the Django admin UI.
- Draft GitHub Actions CI caching strategies to prepare for the first PR.

#### 3.1 Phase 1: Infrastructure & Core Apps (Weeks 1 - 4)
- Week 1: Finalize and merge ``PlaywrightTestCase`` base class with ``addClassCleanup`` async scoping. Implement the ``--playwright`` test runner flag in ``DiscoverRunner``.
- Week 2: Implement GitHub Actions caching for Playwright browsers (``.github/workflows/tests.yml``). Ensure CI execution time parity.
- Week 3: Migrate low-complexity core tests in ``django.contrib.auth``. Replace explicit Selenium waits with Playwright auto-waiting.
- Week 4: Migrate tests in ``django.forms``. Monitor CI stability and eliminate any newly discovered flaky behavior.

#### 3.2 Phase 2: The Admin Lift & Accessibility (Weeks 5 - 8)
- Week 5: Integrate ``axe-playwright-python`` natively into the base class. Begin migration of ``django.contrib.admin`` core navigation and login tests.
- Week 6: Migrate admin change list tests. Replace CDP emulation hacks (``setEmulatedMedia``) with native ``page.emulate_media()``.
- Week 7: Migrate admin form tests and inline widget tests. Replace ``trigger_resize`` hacks with deterministic viewport sizing.
- Week 8: Apply comprehensive Axe checks across migrated admin views. Ensure all CSP log checking is ported to ``page.on('console')``.

#### 3.3 Phase 3: The Long Tail & Deprecation (Weeks 9 - 12)
- Week 9: Migrate ``django.contrib.messages`` and remaining smaller modules.
- Week 10: Migrate ``django.contrib.gis``. Handle any specific edge cases with map rendering. Validate 100% feature parity with the legacy Selenium suite.
- Week 11: Remove ``SeleniumTestCase`` entirely. Delete legacy wait helpers, custom exception handlers for Chrome 113+, and drop Selenium from CI dependencies.
- Week 12: Update Django's internal testing documentation (``docs/internals/contributing/writing-code/unit-tests.txt``). Final review and buffer for community feedback.

### 4. About Me
My name is Vignesh A. I am a student and an active contributor to the Django framework. I have previously authored and merged 6 Pull Requests into Django core, including

- [Fixed #36857 -- Added QuerySet.totally_ordered property.](https://github.com/django/django/pull/20518)

- [Fixed #29257 -- Caught DatabaseError when attempting to close a possibly already-closed cursor.](https://github.com/django/django/pull/20321)

- [Fixed #36112 -- Added fallback in last_executed_query() on Oracle and PostgreSQL.](https://github.com/django/django/pull/20312)

- [Fixed #36030 -- Fixed precision loss in division of Decimal literals on SQLite.](https://github.com/django/django/pull/20309)

- [Fixed #36750 -- Made ordering of M2M objects deterministic in serializers.](https://github.com/django/django/pull/20308)

- [Fixed #36741 -- Linked Model.save() bulk-friendliness discussion from bulk_update() caveats.](https://github.com/django/django/pull/20274)

The Google Summer of Code 2026 programme is an incredible opportunity to deepen my involvement in my favourite web framework by working with the core contributors and community on my earlier contributions. I am very appreciative of the help that the core contributors and community members have given me so far in making my first contributions, and I am looking forward to using my time during the summer to improve the testing infrastructure of Django and provide my services.
