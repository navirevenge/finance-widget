# finance-widget — design: M1 (working widget), M2 (AWS hosting)

## Context
You want a medium iPhone Home Screen widget with three tabs (Balance, Spending, Allocation), a SwiftUI companion app with the same three views, and a small local Go API (`GET /v1/dashboard`). The app and widget share one Swift model and one response format. Allocation uses your real $3,616 split. This plan is the design, broken into GitHub issues.

What I found:
- No finance widget repo exists yet. The repos in `~/Documents/projects 2026` (navi-go, navi-local-ai-context, navirevenge) are unrelated, so this is a new project.
- Full Xcode isn't installed, only the Command Line Tools. Swift 6.1.2, Go 1.25.4 and `gh` work.
- `gh` is logged in as **navirevenge** (the NazWright token is invalid). The repo goes there.

**Scope of the approved work:** create the repo, labels, two milestones and the issues below. No app code is written and no AWS resources are created in this step. CI via GitHub Actions is part of M1. AWS hosting is planned as M2.

## Design

### Repo layout (`navirevenge/finance-widget`, private)
```
contract/dashboard.sample.json   # one fixture both sides test against
api/                             # Go module
  cmd/server/main.go             # :8080, flag -fixture=../contract/dashboard.sample.json
  internal/dashboard/            # types, load+validate, handler, tests
FinanceKit/                      # local Swift package (no Xcode needed for `swift test`)
  Sources/FinanceKit/            # Dashboard model, Money formatting, totals, SnapshotStore, APIClient
  Resources/sample.json          # bundled first-launch data (copy of the contract fixture)
  Tests/FinanceKitTests/
ios/project.yml                  # XcodeGen spec → FinanceWidget.xcodeproj (app + widget extension)
ios/App/                         # SwiftUI app
ios/Widget/                      # WidgetKit + App Intents
README.md
```
Why XcodeGen: the project file stays a short, readable YAML file instead of a machine-generated pbxproj, and it can be regenerated from the command line.

### Response format (shared contract)
```json
{
  "schemaVersion": 1,
  "generatedAt": "2026-11-27T15:00:00Z",
  "dataLabel": "Sample data",
  "balances":   { "accounts":   [{ "id": "chase", "name": "Chase Checking", "balanceCents": 248312 }] },
  "spending":   { "periodLabel": "Nov 1 – Nov 27 (sample)",
                  "categories": [{ "id": "groceries", "name": "Groceries", "amountCents": 41250 }] },
  "allocation": { "periodLabel": "Per biweekly paycheck",
                  "rows": [
    { "id": "chase",    "account": "Chase Checking",      "amountCents": 127500, "purpose": "Bills, groceries, gas" },
    { "id": "rh",       "account": "Robinhood Savings",   "amountCents": 150000, "purpose": "Savings",
      "components": [{ "label": "General savings", "amountCents": 100000 },
                     { "label": "Mac",             "amountCents": 50000 }] },
    { "id": "capone",   "account": "Capital One Checking","amountCents": 59100,  "purpose": "Spending + card payoff" },
    { "id": "varo",     "account": "Varo Checking",       "amountCents": 25000,  "purpose": "Brokerage transfers" } ] }
}
```
- All money is integer cents (`Int` in Swift, `int64` in Go) and is formatted only at display time.
- **Totals are never stored.** Each total is computed by summing the rows. Components are a breakdown of their row and don't count toward the total. That's why changing Capital One to 37500 gives $3,400 with nothing else to edit.
- Go validates the fixture on every load: non-negative amounts, unique ids, and components no larger than their row. An invalid fixture returns 500 with a clear message.
- The views are built from these rows, so adding or renaming an account means editing JSON, not UI code.

### Data flow and widget state
- **Go API** re-reads the fixture on every request. You can edit the fixture without restarting the server.
- **App Group** (`group.<bundle>.finance`) holds two things:
  - `snapshot.json`: the last good dashboard, with source (`api`, `bundledSample`) and `fetchedAt`.
  - `selectedTab` in shared `UserDefaults`.
- **The app owns the network.** On launch, on returning to the foreground and on pull-to-refresh, it fetches `http://127.0.0.1:8080/v1/dashboard` (2 s timeout). On success it writes the snapshot and calls `WidgetCenter.reloadAllTimelines()`. On failure it keeps the snapshot, or the bundled sample on first launch. Info.plist sets `NSAllowsLocalNetworking` so this plain-http call to the local server is allowed.
- **The widget never makes network calls.** The timeline provider reads the snapshot and `selectedTab`. That keeps tab switching instant and lets it work offline (done-criterion 4).
- **Tab tap:** each tab is a `Button(intent: SelectTabIntent(tab:))`. `perform()` writes `selectedTab` to the App Group, and WidgetKit then reloads the timeline from the snapshot.
- **Body tap:** `widgetURL(financewidget://tab/<tab>)`. The app's `onOpenURL` sets its `TabView` selection.
- **Source label** on both surfaces shows where the data came from: "Live · local API · 3:41 PM", "Snapshot · API offline · last updated 3:41 PM" or "Sample data · bundled".
- **Refresh after a fixture edit:** edit the JSON, open the app (or pull to refresh), and the widget updates on its own. The README documents this and the offline behaviour.

## CI with GitHub Actions (in M1, right after the scaffold)
The first workflow lands in issue 2, before any feature code. From then on, every PR is checked automatically.

`.github/workflows/ci.yml` runs on every push and PR to `main`, with three jobs:

| Job | Runner | Runs only when these change | Steps |
|---|---|---|---|
| `api` | `ubuntu-latest` | `api/**`, `contract/**` | `gofmt -l` check, `go vet`, `go test -race ./...` |
| `financekit` | `macos-latest` | `FinanceKit/**`, `contract/**` | `swift test` |
| `ios-build` | `macos-latest` | `ios/**`, `FinanceKit/**` | `brew install xcodegen`, `xcodegen`, `xcodebuild build-for-testing` for an iPhone Simulator |

- **Contract job:** a small check that `FinanceKit/Resources/sample.json` is identical to `contract/dashboard.sample.json`, so the bundled data can't drift from the fixture.
- **Branch protection on `main`:** these checks must pass before a PR can merge.
- **Cost:** a private repo gets 2,000 free Actions minutes a month, and macOS minutes count 10× (about 200 real macOS minutes). That's why the Swift jobs have path filters and Go runs on Linux. Making the repo public makes standard runners free. Running macOS jobs only on PRs, not on every push, is another lever.

## GitHub issues, milestone "M1 – Working widget"
Labels: `ios`, `widget`, `api`, `shared-model`, `ci`, `docs`, `setup`, `aws`, `later`.

| # | Title | Labels | Depends on | Key acceptance |
|---|---|---|---|---|
| 1 | Set up dev environment and repo scaffold | setup | — | Xcode installed and selected (`xcode-select -s`), XcodeGen via brew, repo layout above, `.gitignore`, README skeleton |
| 2 | Set up CI: GitHub Actions for Go, Swift and Xcode builds | ci, setup | 1 | `ci.yml` with the three jobs above; jobs pass on a near-empty repo (placeholder test per module); contract check; branch protection requires the checks; README badge |
| 3 | Define the dashboard response contract and sample fixture | shared-model, api | 1 | `contract/dashboard.sample.json` with the real allocation split and labelled sample balances/spending; schema documented in README |
| 4 | Go API: `GET /v1/dashboard` from fixture | api | 2, 3 | Re-reads the fixture per request, validates it, `-fixture` flag; handler kept separate from `net/http` setup so M2 can wrap it for Lambda; tests: decode, validation errors, allocation total = 361600, the 37500 edit gives 340000; green in CI |
| 5 | FinanceKit: shared Swift model, money formatting, totals | shared-model | 2, 3 | Codable model, `Money` formatter, computed totals (components excluded), bundled sample; tests decode the contract fixture, total = $3,616, 37500 edit gives $3,400; green in CI |
| 6 | Snapshot store and API client | shared-model | 5 | App Group snapshot read/write, source and `fetchedAt`, 2 s timeout, falls back to snapshot then bundled sample; API base URL is configurable (local now, AWS in M2); tests use a stubbed URLProtocol |
| 7 | Xcode project: app and widget targets | ios, setup | 2, 5 | `project.yml` builds the app and widget extension, App Group entitlement on both, ATS local networking, URL scheme `financewidget`; Simulator build green in CI |
| 8 | Companion app: three tabs with deep links | ios | 6, 7 | `TabView` with Balance, Spending and Allocation (names, amounts, purposes, components); source label; pull to refresh; `onOpenURL` routes to the right tab |
| 9 | Medium widget: interactive tabs with App Intents | widget | 6, 7 | `SelectTabIntent`, clear selected-tab styling, 4 allocation rows with no clipping, body `widgetURL` to the matching tab, works with the API stopped |
| 10 | README walkthrough and M1 verification report | docs | 4, 8, 9 | Exact commands (run API, `xcodegen`, open, add widget, $3,400 repro); trace of one tap on "Allocation" from intent to rows; where state lives and why the refresh works this way; table of what passed vs. needs a manual Simulator check |

## Milestone "M2 – Host the API on AWS" (planned now, built after M1)
Everything goes in your project's selected Region, **us-east-2**.

**Recommended architecture:** API Gateway HTTP API → Lambda (Go, `provided.al2023`, arm64) → fixture in a private S3 bucket.
- **Why Lambda, on-demand:** Go is a stated runtime, so this is a regular Lambda function, not the Node.js-only Web Functions option. The traffic is one person's phone, so it should scale to zero and fall within the free tier. An always-on server (App Runner, ECS, EC2) would cost money even while idle.
- **Why an HTTP API in front:** it costs up to ~70% less per request than a REST API, and it supports throttling and authorizers. A Lambda Function URL is simpler, but it offers no throttling, and "open to anyone" (auth type NONE) isn't an acceptable default for finance data.
- **Why S3 for the fixture:** it keeps the M1 promise that editing the fixture needs no recompile. Upload a new JSON file and both surfaces show $3,400 after refresh, with no redeploy.
- **Auth for M2:** a bearer token, checked by a small Lambda authorizer against a value in SSM Parameter Store (SecureString). It's stored in the iOS Keychain, never in the repo. This is enough for one user. Plan a move to Amazon Cognito when Plaid brings real bank data.
- **Infrastructure as code: AWS SAM** (`template.yaml`). It's the least code for one function, one API and one bucket. CDK (TypeScript) is the alternative if you expect the infrastructure to grow a lot. It's more flexible but adds a Node toolchain.

| # | Title | Labels | Key acceptance |
|---|---|---|---|
| 11 | Check AWS plan and service availability | aws, setup | Run `aws freetier get-account-plan-state --profile navi-main`; confirm Lambda, API Gateway, S3 and SSM are available on your plan; note costs and an AWS Budgets alert |
| 12 | Lambda entrypoint for the Go API | aws, api | `cmd/lambda` wraps the same handler from #4 and reads the fixture from S3; local `sam local start-api` works; Go tests still pass |
| 13 | SAM template: HTTP API, Lambda, S3, authorizer | aws | Least-privilege role (read the one S3 object and the one parameter), throttling on the API, CloudWatch log retention set, alarm on 5xx; `sam deploy` to us-east-2 |
| 14 | CD: deploy from GitHub Actions with OIDC (no stored AWS keys) | aws, ci | IAM role trusted only for `repo:navirevenge/finance-widget:ref:refs/heads/main`; `deploy.yml` runs after CI passes on `main`; `sam deploy`; smoke test `curl` with the token |
| 15 | App: switch to the hosted API | ios, aws | Base URL and token configurable (Keychain); source label reads "Live · AWS"; offline/snapshot behaviour unchanged; $3,400 test done by uploading to S3 |
| 16 | M2 README, cost notes and teardown | docs, aws | Deploy/rollback commands, how to rotate the token, monthly cost estimate, `sam delete` teardown and a decision on keeping or deleting resources |

**Later (label `later`, no milestone):** 17 Plaid linking (and Cognito sign-in), 18 Bedrock insights, 19 transaction classification. Each is a one-paragraph placeholder.

Each issue body will contain: goal, scope, out of scope, acceptance checklist (mirroring your done-criteria where relevant), dependencies, and the files it touches.

## Execution steps (after approval)
1. `gh repo create navirevenge/finance-widget --private --description "iPhone finance widget + companion app"` and clone it to `~/Documents/projects 2026/finance-widget`.
2. Commit this design as `docs/M1-design.md`, the only file committed at this point.
3. `gh label create …` for the labels, then create two milestones with `gh api repos/navirevenge/finance-widget/milestones`: "M1 – Working widget" and "M2 – Host the API on AWS".
4. `gh issue create` for issues 1–19 in order, with "Depends on #n" links in each body.
5. Save your help-level preference (High) in `~/.claude/CLAUDE.md`, as your AWS rules allow.
6. Report the issue URLs.

No AWS resources are created in this step. M2 is only planned here.

## Verification
- `gh issue list -R navirevenge/finance-widget --milestone "M1 – Working widget"` shows 10 issues, the M2 milestone shows 6, and the 3 `later` issues have no milestone.
- Each issue renders with its checklist, and its dependency links point to the right numbers.
- Repo is private and contains only the design doc.
