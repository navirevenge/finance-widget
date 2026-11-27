# finance-widget

An iPhone Home Screen widget and companion app for a personal finance dashboard: **Balance**, **Spending** and **Allocation** (how each biweekly paycheck is split across accounts). The widget's tabs switch in place, and tapping the body opens the app on the same tab. A small local Go API serves the data.

Balances and spending are **sample data** in this milestone. Design: [docs/M1-design.md](docs/M1-design.md). Plan: [issues](https://github.com/navirevenge/finance-widget/issues) and [milestones](https://github.com/navirevenge/finance-widget/milestones).

## Repository layout

| Path | What |
|---|---|
| `contract/` | Shared JSON response format and sample fixture |
| `api/` | Go API: `GET /v1/dashboard` |
| `FinanceKit/` | Swift package shared by the app and the widget |
| `ios/` | XcodeGen spec for the app and widget extension |
| `docs/` | Design notes |

## Prerequisites

- macOS with **Xcode** (full app, not just the Command Line Tools), selected with:
  ```sh
  sudo xcode-select -s /Applications/Xcode.app
  ```
  Open Xcode once to accept the license and install the iOS Simulator runtime.
- **XcodeGen**: `brew install xcodegen`
- **Go** 1.25+: `brew install go`

Check your setup:
```sh
xcodebuild -version
xcrun simctl list devices available | grep iPhone
xcodegen --version
go version
```

## Run the API
_Coming in #4._

## Open the project
_Coming in #7._

## Add the widget
_Coming in #9._

## Reproduce the $3,400 test
_Coming in #10._

## How it works
_Walkthrough, where state lives and refresh behaviour: coming in #10._
