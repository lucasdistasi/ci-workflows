# GitHub Workflows

* build-and-test.yml

Runs `./gradlew clean build -x test` and `./gradlew test`

* update-gradle-wrapper.yml

Updates Gradle Wrapper version.

* build-test-and-publish.yml

Runs `./gradlew clean build -x test` and `./gradlew test` and then it publishes to gh-pages the code coverage.  
It requires a token that will expire on 01/01/2026.  
To create a new one, go to [PAT](https://github.com/settings/personal-access-tokens).  
After creating a new one, update the secret `ACTIONS_TOKEN` in each repository that uses this workflow.
