# 🚀 GitHub Workflows

Reusable CI workflows consumed by other microservices. Two categories of workflows live here:

- **Reusable** (`on: workflow_call`) — designed to be invoked from other repositories.
- **Local-only** (no `workflow_call`) — run inside this repo itself and **cannot** be referenced by other repositories.

## <a id="reusable-workflows"></a>🔁 Reusable Workflows

These workflows declare `on: workflow_call` and are meant to be invoked from other repositories via the `uses:` keyword.
They live under `.github/workflows/` and are referenced as `lucasdistasi/ci-workflows/.github/workflows/<file>@<ref>`
(typically `@main` or a tagged release like `@v2.5.0`).

### <a id="check-and-testyml"></a>☕ `check-and-test.yml`

Runs Gradle checks for Java/Gradle-based projects.

- **Trigger:** `workflow_call`
- **Runner:** `ubuntu-26.04`
- **Toolchain:** JDK 25 (Temurin)
- **Command:** `./gradlew check --no-daemon`
- **Inputs / Secrets:** none

**Consumer example:**

```yaml
jobs:
  check-and-test:
    uses: lucasdistasi/ci-workflows/.github/workflows/check-and-test.yml@v2.5.0
```

### <a id="check-and-test-nodeyml"></a>🟩 `check-and-test-node.yml`

Runs install, lint, tests (unit + e2e), and build for Node.js projects managed with pnpm.

- **Trigger:** `workflow_call`
- **Runner:** `ubuntu-26.04`
- **Toolchain:** Node.js 24, pnpm 11.0.9 (via `pnpm/action-setup@v6`)
- **Steps:**
    - `pnpm install --frozen-lockfile`
    - `pnpm lint`
    - `pnpm --if-present test`
    - `pnpm --if-present test:e2e` *(skipped automatically when the script is absent)*
    - `pnpm build`
- **Inputs / Secrets:** none

**Consumer example:**

```yaml
jobs:
  check-and-test:
    uses: lucasdistasi/ci-workflows/.github/workflows/check-and-test-node.yml@v2.5.0
```

### <a id="check-test-and-publishyml"></a>📊 `check-test-and-publish.yml`

Same as `check-and-test.yml`, plus generates a JaCoCo coverage report and publishes it to GitHub Pages (with a generated
badge).

- **Trigger:** `workflow_call`
- **Runner:** `ubuntu-26.04`
- **Toolchain:** JDK 25 (Temurin)
- **Required permissions:** `contents: write` (for deploying to `gh-pages`)
- **Jobs:**
    - `check-and-test` — `./gradlew check jacocoTestReport`, uploads the JaCoCo report as an artifact.
    - `publish` — downloads the report, generates the JaCoCo badge, and deploys the HTML report to the `gh-pages` branch
      via `JamesIves/github-pages-deploy-action@v4`.
- **Inputs / Secrets:** none

**Consumer example:**

```yaml
jobs:
  check-test-and-publish:
    uses: lucasdistasi/ci-workflows/.github/workflows/check-test-and-publish.yml@v2.5.0
```

> ⚠️ The consumer repository must have GitHub Pages enabled with the `gh-pages` branch as the source.

### <a id="send-telegram-messageyml"></a>✈️ `send-telegram-message.yml`

Sends a plain text message to a Telegram chat (and optional topic thread) via the Bot API. This is the building block
used by PR notifications across consumer microservices.

- **Trigger:** `workflow_call`
- **Runner:** `ubuntu-26.04`
- **Inputs:**
    - `message` *(string, required)* — message body sent verbatim.
- **Secrets (must be passed by the caller):**
    - `TELEGRAM_BOT_TOKEN` *(required)*
    - `TELEGRAM_CHAT_ID` *(required)*
    - `TELEGRAM_TOPIC_ID` *(optional)* — message thread id for supergroup topics.

**Consumer example:**

```yaml
jobs:
  notify:
    uses: lucasdistasi/ci-workflows/.github/workflows/send-telegram-message.yml@v2.5.0
    with:
      message: |
        🚀 ${{ github.event.repository.name }}
        📝 ${{ github.event.pull_request.title }}
        🔗 ${{ github.event.pull_request.html_url }}
    secrets:
      TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_GH_PR_BOT_TOKEN }}
      TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
      TELEGRAM_TOPIC_ID: ${{ secrets.TELEGRAM_GH_PR_TOPIC_ID }}
```

<hr>

## <a id="local-only-workflows"></a>🔒 Local-only Workflows

These workflows have **no** `workflow_call` trigger. Per GitHub Actions rules, a workflow that lacks `workflow_call`
cannot be referenced by any other workflow via `uses:` (cross-repo or same-repo) — the call fails schema validation.
That makes them safe to use for in-repo behavior that should not be reusable.

### <a id="notify-pr-telegramyml"></a>🔔 `notify-pr-telegram.yml`

Sends a Telegram notification when a PR is opened, synchronized, or reopened against this repository. Internally it
delegates to `send-telegram-message.yml` via a local call (`./.github/workflows/send-telegram-message.yml`).

- **Trigger:** `pull_request` with types `opened`, `synchronize`, `reopened`
- **Runner:** `ubuntu-26.04`
- **Permissions:** `contents: read`
- **Skipped when:** the PR author is `lucasdistasi` (via `if: github.actor != 'lucasdistasi'`), to avoid
  self-notifications.
- **Required secrets (configured in this repo):** `TELEGRAM_GH_PR_BOT_TOKEN`, `TELEGRAM_CHAT_ID`,
  `TELEGRAM_GH_PR_TOPIC_ID`.

> Other microservices that want PR notifications should call `send-telegram-message.yml` directly from their own
> `pull_request` workflow, not from this wrapper.

<hr>

## <a id="consuming-reusable-workflows"></a>📥 Consuming Reusable Workflows

Consumer microservices invoke these workflows by referencing them with `uses:` and pinning to a ref (branch, tag, or
SHA). **Pinning to a tag (`@v2.5.0`) is the recommended approach for stability** — do not rely on `@main` in production
branches.

The pattern used across the `ld-*` services is a split of two thin wrapper files, one per trigger:

**`.github/workflows/check-and-test.yml`** — runs on every PR, notifies Telegram and runs the checks:

```yaml
name: Notify, Check & Test

on:
  pull_request:
    types: [ opened, synchronize ]

permissions:
  contents: read

jobs:
  notify-by-telegram:
    name: Notify by Telegram
    if: github.actor != 'lucasdistasi'
    uses: lucasdistasi/ci-workflows/.github/workflows/send-telegram-message.yml@v2.5.0
    secrets:
      TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_GH_PR_BOT_TOKEN }}
      TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
      TELEGRAM_TOPIC_ID: ${{ secrets.TELEGRAM_GH_PR_TOPIC_ID }}
    with:
      message: |
        🚀 ${{ github.event.repository.name }}

        📝 ${{ github.event.pull_request.title }}

        👤 ${{ github.actor }}

        📅 ${{ github.event.pull_request.created_at }}

        🔀 ${{ github.event.pull_request.base.ref }} <- ${{ github.event.pull_request.head.ref }}

        🔗 ${{ github.event.pull_request.html_url }}

  call-check-and-test:
    name: Check & Test
    uses: lucasdistasi/ci-workflows/.github/workflows/check-and-test.yml@v2.5.0
```

**`.github/workflows/main.yml`** — runs on every push to `main`, checks only (no PR notification):

```yaml
name: CI

on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  check-and-test:
    uses: lucasdistasi/ci-workflows/.github/workflows/check-and-test.yml@v2.5.0
```

This split keeps PR notifications isolated from `main` builds and lets the `pull_request` workflow carry its own richer
message body (date, branch, author) without affecting post-merge runs.

> 🧪 `ld-cedears-service` adds a third file, `run-selenium-tests.yml`, triggered on `schedule` + `workflow_dispatch`. It
> runs Selenium integration tests locally, parses the result per broker, and then
> forwards a ✅/❌ summary to Telegram via `send-telegram-message.yml@v2.5.0` (with `TELEGRAM_SELENIUM_*` secrets). Use it
> as a reference for any service that wants to ship its own scheduled job and reuse the same notification primitive.

**Consumer-side secret checklist** (when using `send-telegram-message.yml`):

| Secret                     | Required | Purpose                                                      |
|----------------------------|----------|--------------------------------------------------------------|
| `TELEGRAM_GH_PR_BOT_TOKEN` | yes      | Bot token used to authenticate against the Telegram Bot API. |
| `TELEGRAM_CHAT_ID`         | yes      | Target chat / supergroup id.                                 |
| `TELEGRAM_GH_PR_TOPIC_ID`  | no       | Topic thread id inside a supergroup.                         |

These are repository or organization-level secrets on the *caller* side, not on `ci-workflows`.

## <a id="runner-policy"></a>🏃 Runner Policy

All workflows run on **`ubuntu-26.04`** instead of `ubuntu-latest`. The `ubuntu-latest` label points to Ubuntu 24.04
today and is being migrated to Ubuntu 26.04 through October and November 2026, which makes it a moving target: the same
commit can land on a 24.04 runner (green) or on a 26.04 runner (red) depending on which image the job gets. Pinning the
LTS label keeps the OS line stable until the next Ubuntu release is adopted on purpose.

Two things to keep in mind:

- **The LTS line is pinned, the image is not.** GitHub rebuilds the `ubuntu-26.04` image roughly every week, so the
  kernel and the preinstalled toolchain still change over time. Container based tests must not assume a fixed host
  kernel — for example, MongoDB 8.x refuses to boot on kernels 6.19 - 7.0.13 unless
  `GLIBC_TUNABLES=glibc.pthread.rseq=1` is set.
- **Bump the pin deliberately.** When Ubuntu 28.04 becomes the default, update `runs-on`, tag a new release and point
  consumers at the new tag.

## <a id="dependency-updates"></a>⬆️ Dependency Updates

GitHub Actions versions used by these workflows are kept up to date by **Dependabot** (`.github/dependabot.yml`):

- Weekly schedule.
- One open PR at a time, grouping all updates together.
- Assigned and reviewed by `@lucasdistasi`.
