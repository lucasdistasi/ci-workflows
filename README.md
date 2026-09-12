# GitHub Workflows

* build-and-test.yml

Runs `./gradlew build -x test` and `./gradlew test`

* build-test-and-publish.yml

Runs `./gradlew build -x test` and `./gradlew test` and then it publishes to gh-pages the code coverage.

* check-and-test.yml

Runs `./gradlew check` using JDK 25. Used for Gradle-based projects to perform checks and tests.

* check-and-test-node.yml

Runs `pnpm install`, `pnpm lint`, `pnpm --if-present test`, `pnpm --if-present test:e2e`, and `pnpm build` using Node.js 24 and pnpm 11.0.9. Used for Node.js projects. The e2e step is skipped automatically for services that don't define a `test:e2e` script.
