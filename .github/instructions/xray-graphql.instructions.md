---
applyTo: "**"
---

# Access Xray Cloud directly with the REST and GraphQL APIs

Use Xray Cloud's REST authentication endpoint, its GraphQL endpoint for reading and writing test
definitions, and its REST import endpoints for submitting execution results. This instruction is
portable: copy it into another project's `.github/instructions/` directory.

## Configuration and authentication

- Obtain Xray Cloud API credentials (`XRAY_CLIENT_ID` and `XRAY_CLIENT_SECRET`) for the target Xray tenant. Store them in that project's secret manager or environment, never in source control or client-side code. Use a backend or other trusted runtime.
- Select the tenant's `XRAY_REGION` (`us`, `eu`, or `au`) and use matching endpoints:

  | Region | Authentication | GraphQL |
  | --- | --- | --- |
  | `us` | `https://xray.cloud.getxray.app/api/v2/authenticate` | `https://us.xray.cloud.getxray.app/api/v2/graphql` |
  | `eu` | `https://eu.xray.cloud.getxray.app/api/v2/authenticate` | `https://eu.xray.cloud.getxray.app/api/v2/graphql` |
  | `au` | `https://au.xray.cloud.getxray.app/api/v2/authenticate` | `https://au.xray.cloud.getxray.app/api/v2/graphql` |

- Authenticate with `POST` and `Content-Type: application/json`, body `{"client_id":"...","client_secret":"..."}`. The response is a JSON string containing the bearer token, not an object with an `access_token` field. Cache the token only in trusted memory; refresh before expiry (55 minutes is a safe interval) and reauthenticate on expiry.
- Query the GraphQL endpoint with `POST`, `Content-Type: application/json`, `Authorization: Bearer <token>`, and JSON body `{"query":"...","variables":{...}}`. Check HTTP status **and** the GraphQL `errors` array; HTTP 200 can still contain errors or partial `data`. Do not silently treat missing data as an empty result. Never log credentials or tokens.

## Read tests

For numeric Jira issue IDs, use Xray's `issueIds` filter (strings, such as `"10001"`). For Jira issue keys such as `PROJ-123`, use `jql` instead (for example `issue in ("PROJ-123")`). Do not pass keys to `issueIds`. Validate or safely construct JQL from untrusted input; GraphQL variables protect the GraphQL document but do not escape JQL syntax. Limit each page to at most 100 tests, and paginate with `start` until all `total` results are read. For long lists of issue IDs, split the input into batches of at most 100 and query each batch.

```graphql
query ReadXrayTests($issueIds: [String], $jql: String, $start: Int!, $limit: Int!) {
  getTests(issueIds: $issueIds, jql: $jql, start: $start, limit: $limit) {
    total
    results {
      issueId
      testType { name }
      steps {
        action
        data
        result
        customFields { id value }
      }
      unstructured
      gherkin
      folder { path }
      dataset {
        parameters { name }
        rows { Values }
      }
      jira(fields: ["key", "summary", "description", "status", "project", "labels", "components", "fixVersions", "priority"])
    }
  }
}
```

Variables for two numeric issue IDs: `{"issueIds":["10001","10002"],"jql":null,"start":0,"limit":2}`.

Variables for issue keys: `{"issueIds":null,"jql":"issue in (\"PROJ-123\", \"PROJ-124\")","start":0,"limit":2}`.

The API returns `{ "data": { "getTests": { "total": ..., "results": [...] } } }`, not an array of flattened tests. `jira` holds the requested Jira fields (for example `jira.key` and `jira.summary`); Xray-specific fields are on each result (`issueId`, `testType.name`, `steps`, `folder.path`, `gherkin`, `unstructured`, `dataset`). Manual tests have steps, Generic tests can have `unstructured`, and Cucumber tests can have `gherkin`. Fields can be absent or null depending on the test and permissions. `dataset.rows` uses the schema field `Values` with a capital V.

## Minimal Node.js example (Node 18+)

Use the runtime's native `fetch`. Set `XRAY_CLIENT_ID` and `XRAY_CLIENT_SECRET` before running, and set `XRAY_REGION` if the tenant is not in the US:

```js
const endpoints = {
  us: ['https://xray.cloud.getxray.app/api/v2/authenticate', 'https://us.xray.cloud.getxray.app/api/v2/graphql'],
  eu: ['https://eu.xray.cloud.getxray.app/api/v2/authenticate', 'https://eu.xray.cloud.getxray.app/api/v2/graphql'],
  au: ['https://au.xray.cloud.getxray.app/api/v2/authenticate', 'https://au.xray.cloud.getxray.app/api/v2/graphql'],
};

const region = (process.env.XRAY_REGION || 'us').toLowerCase();
if (!endpoints[region]) throw new Error(`Invalid XRAY_REGION: ${region}`);
if (!process.env.XRAY_CLIENT_ID || !process.env.XRAY_CLIENT_SECRET) {
  throw new Error('XRAY_CLIENT_ID and XRAY_CLIENT_SECRET are required');
}
const [authUrl, graphqlUrl] = endpoints[region];

async function postJson(url, body, headers = {}) {
  const response = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', ...headers },
    body: JSON.stringify(body),
  });
  if (!response.ok) throw new Error(`Xray request failed: HTTP ${response.status}`);
  return response.json();
}

async function readTests(issueIds) {
  const token = await postJson(authUrl, {
    client_id: process.env.XRAY_CLIENT_ID,
    client_secret: process.env.XRAY_CLIENT_SECRET,
  });
  if (typeof token !== 'string' || !token) throw new Error('Invalid Xray auth response');

  const query = `
    query ReadTests($issueIds: [String], $start: Int!, $limit: Int!) {
      getTests(issueIds: $issueIds, start: $start, limit: $limit) {
        total
        results {
          issueId
          testType { name }
          steps { action data result }
          unstructured
          gherkin
          jira(fields: ["key", "summary"])
        }
      }
    }
  `;
  const tests = [];
  for (let start = 0; ; start += 100) {
    const response = await postJson(
      graphqlUrl,
      { query, variables: { issueIds, start, limit: 100 } },
      { Authorization: `Bearer ${token}` },
    );
    if (response.errors?.length) {
      throw new Error(`Xray GraphQL errors: ${response.errors.map((e) => e.message).join('; ')}`);
    }
    const page = response.data?.getTests;
    if (!page || !Array.isArray(page.results) || !Number.isInteger(page.total)) {
      throw new Error('Unexpected Xray getTests response');
    }
    tests.push(...page.results);
    if (tests.length >= page.total) return tests;
    if (page.results.length === 0) throw new Error('Xray returned an empty page before total was reached');
  }
}

readTests(['10001', '10002']).then(console.log).catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

## Import execution results

Results are submitted to REST import endpoints, not to GraphQL. Authenticate exactly as above and
send the same bearer token. The endpoint is chosen by report format:

| Format | Endpoint |
| --- | --- |
| Xray JSON | `/api/v2/import/execution` |
| JUnit XML | `/api/v2/import/execution/junit` |
| TestNG XML | `/api/v2/import/execution/testng` |
| NUnit, xUnit, Robot, Cucumber, Behave | `/api/v2/import/execution/<format>` |

Every format also has a `/multipart` variant. Use the base endpoint with query parameters for simple
submissions, and the multipart variant when Test Execution fields must be controlled.

### JUnit XML

`POST /api/v2/import/execution/junit` with `Content-Type: text/xml` and the report as the raw body.

| Parameter | Purpose |
| --- | --- |
| `projectKey` | Project in which to create the Test Execution and any new tests. Used only when `testExecKey` is absent. |
| `testExecKey` | Import into an existing Test Execution instead of creating one. |
| `testPlanKey` | Add the tests to this Test Plan if they are not already in it. |
| `testEnvironments` | Semicolon-separated list of environments. |
| `revision` | Source or documentation version under test. |
| `fixVersion` | Fix Version for the Test Execution. Accepts one value only. |

```bash
curl -H "Content-Type: text/xml" -X POST \
  -H "Authorization: Bearer $TOKEN" \
  --data @results.xml \
  "https://xray.cloud.getxray.app/api/v2/import/execution/junit?testExecKey=PROJ-123"
```

The response is `{"id":"...","key":"...","self":"..."}` for the Test Execution.

### How a testcase binds to a test

Xray resolves each `<testcase>` in this order:

1. A `test_id` property on the `testcase` element. Wins over `test_key`.
2. A `test_key` property on the `testcase` element.
3. The **Definition** field of an Unstructured test, compared against the `classname` and `name`
   attributes concatenated. When `classname` is absent, the `testsuite` name is used instead.

An explicit `test_id` or `test_key` that does not resolve raises an error. Definition matching that
finds nothing **silently creates a new test**, so a mismatch surfaces as duplicates rather than as a
failure. Prefer explicit properties when the mapping is already known:

```xml
<testcase classname="com.example.LoginTest" name="testValidLogin" time="1.358">
  <properties>
    <property name="test_key" value="PROJ-456"/>
  </properties>
</testcase>
```

Definition matching searches the target project first, then every other Jira project, before
creating anything.

### Constraints worth knowing

- Auto-provisioned tests are created with the **Generic** test type. Selecting a different type at
  creation time is supported only by the Xray JSON format, through `testInfo.type`.
- The separator Xray inserts between `classname` and `name` is documented for NUnit (`.`) but not
  for JUnit. Verify empirically against the tenant before hard-coding a Definition string.
- The `/multipart` variants take no query parameters. They take `results`, `info`, and `testInfo`
  file parts, where `info` sets Test Execution fields and `testInfo` sets fields on newly created
  tests. Issue type and summary in `testInfo` are ignored.
- Test types have a *kind*: `Steps`, `Unstructured`, or `Gherkin`. The Definition field belongs to
  the Unstructured kind, so any custom Unstructured type has it, whatever the type is named.

## General

For production, cache and refresh the token, handle HTTP 401 by reauthenticating, and retry HTTP
429 and transient failures with bounded backoff according to response headers. Do not make unbounded
parallel requests. Keep GraphQL query selection small: Xray enforces limits on nested result nodes
and resolved fields; split large or deep reads into multiple paginated queries rather than
requesting every relationship at once. Verify the tenant's API access and Jira project permissions
if tests are missing. Never log credentials or tokens.
