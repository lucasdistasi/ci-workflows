# GitHub Workflows

* build-and-test.yml

Runs `./gradlew build -x test` and `./gradlew test`

* build-test-and-publish.yml

Runs `./gradlew build -x test` and `./gradlew test` and then it publishes to gh-pages the code coverage.  

* check-and-test.yml

Runs `./gradlew check` using JDK 25. Used for Gradle-based projects to perform checks and tests.

* check-and-test-node.yml

Runs `pnpm install`, `pnpm lint`, `pnpm test --if-present`, and `pnpm build` using Node.js 24 and pnpm 10.33.3. Used for Node.js projects.
