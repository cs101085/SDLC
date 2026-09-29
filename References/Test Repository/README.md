# Compass-Automation — Repository Context

> Compass-Automation is a Java 21 + Maven UI test-automation project that drives the Compass
> web application with Playwright for Java, organizes tests with TestNG suites,
> supplies test data from JSON files mapped to model classes with Jackson, seeds and cleans
> database state through JDBC against PostgreSQL or Oracle, and publishes an Extent Spark
> HTML report. Compass-Automation produces test results and reports only; Compass-Automation
> contains no application/production code.

| Field | Value |
| --- | --- |
| Analysis root | `Test Repository/` |
| Analyzed on | `2026-09-25` |
| Branch / commit | `[unknown]` — no Git metadata was readable from the analysis root |
| Maven coordinates | `com.smartwork.compass:Compass-Automation:0.0.1-SNAPSHOT` |
| Primary language | `Java 21` (`maven.compiler.source` and `maven.compiler.target` are `21`) |
| Build / dependency manager | `Maven` (`pom.xml`) |
| Shape | Single Maven module, `24` TestNG test classes, `30` page objects |
| Paths in this document | Relative to the analysis root |

## How to use this document

- Trust the facts below. Search the codebase only when a fact is missing, marked `[unknown]`, or contradicted by code.
- If the code contradicts this document, the code wins.
- Status markers: `[from-config]` read from `pom.xml`, a TestNG suite file, or `config.properties`; `[from-docs]` claimed in project documentation and unconfirmed; `[inferred]` deduced from code and not executed; `[unknown]` open question.
- No command in this document was executed. Treat every command as unvalidated.
- `.github/copilot-instructions.md` holds the authoring rules for agents working in Compass-Automation. Read `.github/copilot-instructions.md` before generating Playwright Java code.

## Purpose and scope

- Automates end-to-end UI regression and smoke checks for Compass features such as Edit Meter, Location Summary, Heat Map, Line Graph, Interval Bar Graph, VEE Processor, Manage KPI, Rule Workflow, Virtual Meter, and User Permission.
- Seeds database fixtures before a scenario and removes seeded rows after the scenario through `DbCleanupManager`.
- Produces an Extent Spark HTML report plus failure screenshots under `test-output/`.
- Does not contain Compass application source code.
- Does not define JavaScript or TypeScript Playwright tests; `.github/copilot-instructions.md` forbids generating them.
- Does not ship CI pipeline definitions; no workflow file exists under `.github/`.

## Technology stack

| Layer | Technology | Version | Declared in |
| --- | --- | --- | --- |
| Language | Java | `21` | `pom.xml` |
| Browser automation | `com.microsoft.playwright:playwright` | `1.57.0` | `pom.xml` |
| Test runner | `org.testng:testng` | `7.12.0` | `pom.xml` |
| Reporting | `com.aventstack:extentreports` | `5.1.2` | `pom.xml` |
| PostgreSQL driver | `org.postgresql:postgresql` | `42.7.10` | `pom.xml` |
| Oracle driver | `com.oracle.database.jdbc:ojdbc11` | `23.26.2.0.0` | `pom.xml` |
| JSON binding | `com.fasterxml.jackson.core:jackson-databind` | `2.18.6` | `pom.xml` |
| Excel / export checks | `org.apache.poi:poi-ooxml` | `5.2.3` | `pom.xml` |
| Suite execution | `maven-surefire-plugin` | `3.2.5` | `pom.xml` |
| Playwright CLI launcher | `exec-maven-plugin` | `3.6.3` | `pom.xml` |

## Directory map

| Path | Contains | Add new code here when… |
| --- | --- | --- |
| `src/main/java/pages/` | Page objects extending `BasePage`, one class per Compass feature | Adding or changing UI interaction logic and locators |
| `src/main/java/models/` | Plain public-field data classes bound from JSON by Jackson | A new JSON test-data shape is required |
| `src/main/java/utils/` | `ConfigReader`, `WaitHelper`, `Timeouts`, `DataProviderUtil`, `DataSource`, `JsonDataReader`, `UniqueDataResolver`, `DatabaseHelper`, `DbCleanupManager`, `ExtentManager`, `ScreenshotUtil` | Adding cross-cutting framework support |
| `src/main/java/database/` | `DatabaseFixtureLoader`, the JSON-driven fixture engine | Extending declarative fixture behavior |
| `src/main/resources/` | `config.properties` with environment, browser, timeout, and cleanup keys | A new configuration key is required |
| `src/test/java/base/` | `BaseTest`, the TestNG lifecycle and login harness | Changing setup, teardown, or reporting hooks |
| `src/test/java/tests/` | TestNG test classes, one per feature, all extending `BaseTest` | Adding a scenario |
| `src/test/java/assertion/` | Assertion classes extending `AssertionHelper` | Adding verification logic |
| `src/test/java/database/` | `MeterDatabase`, hand-written SQL helper | A fixture needs SQL that the JSON engine cannot express |
| `src/test/java/data/` | `TestData`, legacy hard-coded TestNG data providers | Never; prefer `@DataSource` with JSON |
| `src/test/resources/TestData/` | `*.json` test data and `databaseFixtures` definitions | Adding or editing scenario data |

Generated output that must never be hand-edited: `test-output/` (Extent report, `screenshots/`, `downloads/`) and `target/`. Both paths are listed in `.gitignore`.

## Entry points

| Entry point | File and symbol | Invoked by |
| --- | --- | --- |
| Full cross-browser suite | `testNg.xml`, suite `Cross Browser Suite` | `maven-surefire-plugin` configuration in `pom.xml` `[from-config]` |
| Smoke suite | `smoketest.xml`, suite `Smoke Test Suite`, group `smoke` | Explicit `-DsuiteXmlFile` override `[inferred]` |
| Regression suite | `regressiontest.xml`, suite `Regression Test Suite`, group `regression` | Explicit `-DsuiteXmlFile` override `[inferred]` |
| Test lifecycle | `src/test/java/base/BaseTest.java` | TestNG annotations `@BeforeSuite`, `@BeforeClass`, `@BeforeMethod`, `@AfterMethod`, `@AfterClass`, `@AfterSuite` |
| Data provider | `src/main/java/utils/DataProviderUtil.java`, provider name `jsonData` | Any `@Test` declaring `dataProvider = "jsonData"` |
| Browser installation | `com.microsoft.playwright.CLI` via `exec-maven-plugin` | `mvn exec:java` `[from-config]` |

## Commands

| Purpose | Command | Run from | Source | Status |
| --- | --- | --- | --- | --- |
| Install Playwright browsers | `mvn exec:java -D"exec.args=install"` | analysis root | `exec-maven-plugin` `mainClass` in `pom.xml` | `[from-config]` |
| Compile sources | `mvn clean compile` | analysis root | Maven default lifecycle | `[inferred]` |
| Run the configured suite | `mvn test` | analysis root | `suiteXmlFiles` entry in `pom.xml` | `[from-config]` |
| Run the smoke suite | `mvn test -DsuiteXmlFile=smoketest.xml` | analysis root | `smoketest.xml` | `[inferred]` |
| Run the regression suite | `mvn test -DsuiteXmlFile=regressiontest.xml` | analysis root | `regressiontest.xml` | `[inferred]` |
| Run one test class | `mvn test -Dtest=EditMeterTest` | analysis root | Maven Surefire `-Dtest` selector | `[inferred]` |
| Select the Oracle environment | `mvn test -Dactive.env=env2` | analysis root | `System.getProperty("active.env", ...)` in `ConfigReader` | `[from-config]` |

Ordering requirements:

- Install Playwright browsers once before the first `mvn test` run, because `Playwright.create()` in `BaseTest.beforeClass` launches a downloaded browser binary. `[inferred]`
- The target Compass environment and its database must be reachable before a run, because `BaseTest.setUp` logs in through the UI and `DatabaseHelper.getConnection` opens a JDBC connection. `[inferred]`

## Configuration model

`src/main/resources/config.properties` is loaded once by the static block in `ConfigReader`. Values are read by key; this document lists key names only and never values.

| Key | Purpose |
| --- | --- |
| `active.env` | Default environment; accepted values are `env1` and `env2` only |
| `<env>.baseUrl` | Login page address for the selected environment |
| `<env>.username`, `<env>.password` | Compass UI credentials used by `BaseTest.setUp` |
| `<env>.db.url`, `<env>.db.username`, `<env>.db.password`, `<env>.db.driver` | JDBC settings used by `DatabaseHelper.getConnection` |
| `browser`, `headless`, `slowMo` | Playwright launch defaults used by `BaseTest.beforeClass` |
| `timeout.short`, `timeout.default`, `timeout.long`, `timeout.aiResponse`, `timeout.polling` | Millisecond values exposed as `Timeouts` constants |
| `db.cleanup.enabled`, `db.runTagPrefix` | Cleanup switch and run-tag prefix used by `DbCleanupManager` |

Resolution order for the active environment:

1. The `environment` suite parameter in the TestNG suite file, applied by `BaseTest.beforeSuite` through `ConfigReader.setActiveEnv`. `[from-config]`
2. The `-Dactive.env=<env1|env2>` system property. `[from-config]`
3. The `active.env` key in `config.properties`. `[from-config]`

`ConfigReader.getEnv("baseUrl")` resolves the key `<activeEnv>.baseUrl`. `ConfigReader.get` throws when a key is missing or blank.

## Architecture and data flow

Primary flow for one test method, end to end:

1. `BaseTest.beforeSuite` sets the active environment and initializes the shared `ExtentReports` instance through `ExtentManager.getInstance`.
2. `BaseTest.beforeClass` resolves the browser from the `browserName` suite parameter, then from `config.properties`, then falls back to `chromium`, and launches one Playwright browser per test class.
3. `DataProviderUtil.provide` reads the `@DataSource` annotation of the test method, loads the named file from `src/test/resources/TestData/`, and returns model objects.
4. `JsonDataReader.getData` parses the requested array, calls `UniqueDataResolver.resolveJson` to expand placeholders, and converts the array to `List<T>` with Jackson.
5. `BaseTest.setUp` creates a fresh `BrowserContext` and `Page`, starts a `DbCleanupManager` run tag, then logs in through `pages.Login`.
6. The test method seeds data with `DatabaseFixtureLoader.insert(...)` or `MeterDatabase.insertMeter(...)`, navigates with `pages.SideMenu`, acts through a feature page object, and verifies through an assertion class.
7. `BaseTest.tearDown` executes registered cleanup steps in reverse registration order, logs pass, fail, or skip to Extent, attaches a failure screenshot from `ScreenshotUtil`, and closes the page and context.
8. `BaseTest.afterClass` closes the browser and the Playwright instance; `BaseTest.afterSuite` flushes the Extent report to `test-output/CompassTestReport.html`.

Boundaries that must not be crossed:

- Tests never build raw locators; locators live in page objects under `src/main/java/pages/`. `[inferred]`
- Page objects never assert; verification lives in `src/test/java/assertion/`. `[inferred]`
- Tests never open a JDBC connection directly; database access goes through `DatabaseHelper`, `DatabaseFixtureLoader`, or `MeterDatabase`. `[inferred]`
- Tests never duplicate iframe chains; frame access goes through the `BasePage` accessors. `[from-docs]`

## Compass iframe model

Compass nests frames, so every locator must be scoped to the frame that owns the element.

| `BasePage` accessor | Frame chain | Typical use |
| --- | --- | --- |
| `headerFrame` | `#root_application_container_header_iframe` | Global application header |
| `contentFrame` | `#root_application_container_iframe` → `frame[name='content']` | Parent of menu and view frames |
| `getMenuFrame()` | `contentFrame` → `frame[name='menu']` | Left navigation menu |
| `getViewFrame()` | `contentFrame` → `frame[name='view']` | Main feature page |
| `getFormFrame()` | `getViewFrame()` → `iframe.jsUtils-dialog-iframe` | Dialog opened inside the view frame |
| `getForm1Frame()` | top-level `iframe.jsUtils-dialog-iframe` (last) | Dialog overlaying the whole application |
| `getVisualFrame()` | `getViewFrame()` → `iframe.visualIframe` (last) | Visual iframe used by Instant Insight |

## Key abstractions

| Abstraction | Defined in | Contract |
| --- | --- | --- |
| `BaseTest` | `src/test/java/base/BaseTest.java` | Owns Playwright lifecycle, login, Extent logging, screenshots, and database cleanup; every test class extends `BaseTest` |
| `BasePage` | `src/main/java/pages/BasePage.java` | Abstract parent of every page object; exposes frame accessors and shared actions such as `clickButtonByName`, `clickSubmit`, `applyColumnFilter` |
| `AssertionHelper` | `src/test/java/assertion/AssertionHelper.java` | Extends `BasePage`; wraps Playwright `assertThat` helpers such as `assertToastMessage` and `assertVisible` |
| `@DataSource` | `src/main/java/utils/DataSource.java` | Method annotation with `file`, `arrays` (default `validData`), `asList`, and `model`; consumed by the `jsonData` provider |
| `DataProviderUtil` | `src/main/java/utils/DataProviderUtil.java` | Supports three modes: full list when `asList=true`, one model per row for a single array, and side-by-side zipping for multiple arrays |
| `UniqueDataResolver` | `src/main/java/utils/UniqueDataResolver.java` | Expands JSON placeholders such as `${unique:PREFIX:scope}`, `${date:plusMonths1:yyyyMMdd}`, `${time:plusHours1:HH:mm:ss}`, `${month:current:MM}`, `${meterId:scope}` |
| `DatabaseFixtureLoader` | `src/main/java/database/DatabaseFixtureLoader.java` | Executes `databaseFixtures` blocks declared in a JSON data file, validates SQL identifiers, supports `returning`/`contextField` chaining, and registers cleanup |
| `DbCleanupManager` | `src/main/java/utils/DbCleanupManager.java` | Thread-local run tag and cleanup stack; `executeCleanup` runs steps in reverse order so child rows are removed before parent rows |
| `Timeouts` | `src/main/java/utils/Timeouts.java` | Constants `SHORT`, `DEFAULT`, `LONG`, `AI_RESPONSE`, `POLLING` sourced from `config.properties` |
| `WaitHelper` | `src/main/java/utils/WaitHelper.java` | Explicit waits: `waitForVisible`, `waitForHidden`, `waitForAttached`, `waitForPageLoad`, `waitForTextVisible`, `waitForClickable` |

## Test data format

Each file in `src/test/resources/TestData/` is a JSON object with two kinds of top-level keys:

- `databaseFixtures`: named fixture objects with `operation`, `table`, `columns`, optional `types`, optional `where`, optional `cleanup`, and optional `expectedAffectedRows`.
- Data arrays such as `validEditMeterData` or `validData`: arrays of objects whose field names match the public fields of the bound model class.

A test binds data and a fixture together, as in `EditMeterTest.testEditMeter`, which annotates the method with `@DataSource(file = "MeterData.json", arrays = {"validEditMeterData"}, model = MeterData.class)` and then calls `DatabaseFixtureLoader.insert("MeterData.json", "meterInsert", data)`.

## Conventions

- Name a test class `<Feature>Test`, its page object `<Feature>`, and its assertion class `<Feature>Assertion`.
- Tag every `@Test` with `groups = "regression"`, `groups = "smoke"`, or both, because `regressiontest.xml` and `smoketest.xml` select by group.
- Register a new test class in `testNg.xml` and in the suite files that must execute the class.
- Instantiate page objects and assertion classes in a class-level `@BeforeMethod` named `initPages`, following `EditMeterTest`.
- Supply data through `@DataSource` plus the `jsonData` provider; do not hard-code business data in a test method.
- Prefer role, label, placeholder, stable text, then application-owned `id` when choosing a locator; avoid absolute or index-based XPath.
- Use `WaitHelper` methods with `Timeouts` constants; do not call `Thread.sleep()` and do not hard-code millisecond literals.
- Keep credentials and URLs in `config.properties`; never hard-code an environment value, credential, token, or cookie in Java or JSON.
- Log narrative steps with `test.info(...)` and call `captureStep(...)` when a screenshot adds diagnostic value.

## External dependencies and integrations

| Integration | Purpose | Config key names | Local substitute |
| --- | --- | --- | --- |
| Compass web application | System under test; reached through the login page | `<env>.baseUrl`, `<env>.username`, `<env>.password` | None; a reachable Compass instance is required `[inferred]` |
| PostgreSQL (`env1`) | Fixture seeding and cleanup | `env1.db.url`, `env1.db.username`, `env1.db.password`, `env1.db.driver` | None identified `[unknown]` |
| Oracle (`env2`) | Fixture seeding and cleanup | `env2.db.url`, `env2.db.username`, `env2.db.password`, `env2.db.driver` | None identified `[unknown]` |
| Playwright browser binaries | Chromium, Firefox, WebKit, and the `msedge` channel | `browser`, `headless`, `slowMo` | Installed locally by `mvn exec:java -D"exec.args=install"` `[from-config]` |

## Definition of done

1. The new or changed test compiles under Java 21.
2. The test is registered in `testNg.xml` and in every suite file that must run the test.
3. Every `@Test` carries the group that matches the suite intent.
4. Locators live in a page object and are scoped to the correct frame accessor.
5. Assertions live in an assertion class that extends `AssertionHelper`.
6. Any row created by the test is removed by a fixture `cleanup` block or by `DbCleanupManager.registerCleanup`.
7. No credential, host, URL, token, or cookie appears in Java, JSON, or Markdown.
8. The targeted suite run leaves `test-output/CompassTestReport.html` with the expected results.

CI equivalent: none. No workflow file exists under `.github/`, so validation is manual. `[inferred]`

## Gotchas

- `pom.xml` configures `<suiteXmlFile>testng.xml</suiteXmlFile>`, but the file in the analysis root is named `testNg.xml`. Resolution succeeds on case-insensitive Windows filesystems and fails on case-sensitive Linux filesystems, where Surefire reports a missing suite file. `[inferred]`
- `regressiontest.xml` lists `tests.IntelligentSegmentationTest`, and no such class exists under `src/test/java/tests/`. TestNG fails suite initialization when a listed class cannot be loaded. `[inferred]`
- `tests.InstantInsightTest` exists and is absent from `testNg.xml`, so `mvn test` with the default suite skips Instant Insight coverage. `[from-config]`
- `src/test/java/tests/test.java` and `src/test/java/assertion/testAssertion.java` are early prototypes. `test.java` calls `ConfigReader.get("baseUrl")`, `ConfigReader.get("username")`, and `ConfigReader.get("password")`, and `config.properties` defines only environment-prefixed keys, so those calls throw at runtime. Follow `EditMeterTest` instead. `[inferred]`
- `testNg.xml` declares `parallel="classes" thread-count="3"`, so page objects and assertion classes must not hold mutable static state. `DbCleanupManager` isolates per thread with `ThreadLocal`; `ExtentReports` is shared deliberately. `[from-config]`
- `config.properties` is committed and holds environment credentials in plain text. Never echo, screenshot, paste, or commit its values, and never print a resolved connection string into a report. `[inferred]`
- `DatabaseHelper.getConnection` prints the JDBC URL to standard output on every connection, so console logs may expose host details. `[inferred]`
- `BaseTest.tearDown` fails a test when database cleanup fails, even if the scenario itself passed. A cleanup exception is attached as a suppressed throwable when the test already failed. `[inferred]`
- One browser is launched per test class and one context per test method; a test that leaks a page or context also leaks a browser process. `[inferred]`
- `test-output/` is listed in `.gitignore` as `/test-output/` and currently holds a `downloads/` folder; treat every file below `test-output/` as disposable. `[from-config]`

## Where to change X

| Task | Touch these files | Then run |
| --- | --- | --- |
| Add a scenario for an existing feature | `src/test/java/tests/<Feature>Test.java`, `src/test/resources/TestData/<Feature>Data.json`, suite files | `mvn test -Dtest=<Feature>Test` |
| Add a new feature area | New page object in `src/main/java/pages/`, new model in `src/main/java/models/`, new assertion class, new test class, new JSON file, entries in `testNg.xml` and `regressiontest.xml` | `mvn test -DsuiteXmlFile=regressiontest.xml` |
| Fix a broken locator | The owning class in `src/main/java/pages/` | `mvn test -Dtest=<Feature>Test` |
| Change a timeout | `src/main/resources/config.properties`, consumed through `src/main/java/utils/Timeouts.java` | `mvn test` |
| Add or change a database fixture | `databaseFixtures` in the relevant `src/test/resources/TestData/*.json` | `mvn test -Dtest=<Feature>Test` |
| Switch environment | `active.env` in `src/main/resources/config.properties`, the `environment` suite parameter, or `-Dactive.env` | `mvn test -Dactive.env=env2` |
| Change report title or output path | `src/main/java/utils/ExtentManager.java` | `mvn test` |

## Glossary

| Term | Definition |
| --- | --- |
| Compass | The web application under test; the address is supplied by the `<env>.baseUrl` configuration key |
| ODS | Operational Data Store; the Compass database schema holding meters, reads, and rules |
| VEE | Validation, Estimation, and Editing; the meter-data quality process covered by `VeeProcessorTest` |
| TOU | Time Of Use; rate windows covered by `ManageTOUWindowTest` |
| AMI | Advanced Metering Infrastructure; covered by `AMIServiceLevelTest` |
| KPI | Key Performance Indicator; dashboards covered by `ManageKPITest` and `ManageKpiDashboardTest` |
| Fixture | A `databaseFixtures` entry in a JSON data file that inserts or deletes rows for one scenario |
| Run tag | Unique per-test identifier built by `DbCleanupManager` from `db.runTagPrefix`, the test name, and a UUID fragment |
| Page object | A class under `src/main/java/pages/` that owns locators and actions for one Compass screen |

## Further reading

- [`.github/copilot-instructions.md`](.github/copilot-instructions.md): mandatory authoring rules, iframe map, locator priority, waiting rules, and the required Playwright MCP analysis output format.
- [`pom.xml`](pom.xml): dependency versions, Java level, and the Surefire suite binding.
- [`testNg.xml`](testNg.xml): default suite, parallelism, and the `environment` and `browserName` parameters.
- [`src/test/java/base/BaseTest.java`](src/test/java/base/BaseTest.java): the full test lifecycle in annotation order.
- [`src/main/java/database/DatabaseFixtureLoader.java`](src/main/java/database/DatabaseFixtureLoader.java): the JSON fixture contract and its validation rules.
- [`src/main/java/utils/UniqueDataResolver.java`](src/main/java/utils/UniqueDataResolver.java): the list of supported JSON placeholders.

## Unknowns and unverified claims

| Claim or question | Status | How to resolve |
| --- | --- | --- |
| Branch and commit described by this document | `[unknown]` | Run `git rev-parse --abbrev-ref HEAD` and `git rev-parse HEAD` in the analysis root |
| Every command listed above | `[inferred]` or `[from-config]`, never executed | Run each command locally and record the outcome |
| Does `tests.IntelligentSegmentationTest` exist elsewhere, or is `regressiontest.xml` stale? | `[unknown]` | Search the repository history for the class name and correct `regressiontest.xml` |
| Is the `testng.xml` versus `testNg.xml` name mismatch intentional? | `[unknown]` | Run `mvn test` on a case-sensitive filesystem and observe Surefire's suite resolution |
| Are `test.java` and `testAssertion.java` retained deliberately? | `[unknown]` | Confirm with the framework owner before removing either file |
| Is a disposable local database available for fixture runs? | `[unknown]` | Confirm with the environment owner which database instances may be seeded |
| Do smoke-tagged methods exist in every class listed by `smoketest.xml`? | `[unknown]` | Run `mvn test -DsuiteXmlFile=smoketest.xml` and check the selected method count |
