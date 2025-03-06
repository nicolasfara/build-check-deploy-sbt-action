# Build Check Deploy SBT Action

This GitHub Action automates the process of building, checking, and deploying Scala projects using SBT.

## Features

- **Build**: Compiles the Scala project.
- **Check**: Runs tests and checks code quality.
- **Deploy**: Deploys the project to the specified environment.

## Usage

To use this action, add the following to your workflow configuration file:

```yaml
```yaml
name: Example Workflow

on: [push]

jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Build, Check, Deploy
        uses: nicolasfara/build-check-deploy-sbt-action@v1
        with:
          should-run-codecov: false
          check-command: sbt +test
          should-deploy: true
          maven-central-username: ${{ secrets.MAVEN_CENTRAL_USERNAME }}
          maven-central-password: ${{ secrets.MAVEN_CENTRAL_PASSWORD }}
          signing-key: ${{ secrets.SIGNING_KEY }}
          signing-password: ${{ secrets.SIGNING_PASSWORD }}
```

## Inputs

| Name                     | Description                                                                 | Required | Default        |
|--------------------------|-----------------------------------------------------------------------------|----------|----------------|
| `pre-build-command`      | A command to be executed before the build phase (but after configuring the JVM) | false    | `true`         |
| `build-command`          | The command executed in the build phase                                      | false    | `sbt +package` |
| `check-command`          | The command executed in the check phase                                      | false    | `sbt +test`    |
| `clean-command`          | The command executed in the final clean phase                                | false    | `true`         |
| `retries-on-failure`     | How many times every command should be retried before giving up              | false    | `2`            |
| `wait-between-retries`   | How many seconds to wait after a step failure before attempting a new run    | false    | `5`            |
| `codecov-directory`      | The location where CodeCov searches for coverage information                 | false    | `build`        |
| `codecov-token`          | The codecov token for this repository                                        | false    | `""`           |
| `deploy-command`         | The condition triggering a deploy run                                        | false    | `true`         |
| `java-distribution`      | The Java distributor to use                                                  | false    | `temurin`      |
| `java-version`           | The Java version to use                                                      | false    | `17`           |
| `should-run-codecov`     | True if the action should send coverage results to codecov.io                | false    | `true`         |
| `should-deploy`          | True if the deploy operation should be executed                              | false    | `false`        |
| `maven-central-username` | Username for Maven Central that will be exposed in the deployment step as the environment variables MAVEN_CENTRAL_PASSWORD | false    | N/A            |
| `working-directory`      | Location where the repository should will be cloned                          | false    | `.`            |
| `github-token`           | The GitHub token, it will be exposed in the deployment step as the environment variable GITHUB_TOKEN | false    | N/A            |
| `maven-central-password` | Password for OSSRH / Maven Central, it will be exposed in the deployment step as the environment variables MAVEN_CENTRAL_PASSWORD | false    | N/A            |
| `maven-central-repo`     | URL for OSSRH / Maven Central Repository, it will be exposed in the deployment step as the environment variables MAVEN_CENTRAL_REPO | false    | `https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/` |
| `signing-key`            | ASCII-armored base64 signing key, it will be exposed in the deployment step as the environment variable | false    | N/A            |
| `signing-password`       | Password for the signing key, it will be exposed in the deployment step as the environment variables SIGNING_PASSWORD | false    | N/A            |

## Example

```yaml
name: CI/CD Process
on:
  workflow_call:
  workflow_dispatch:

jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        os: [ windows-2022, macos-14, ubuntu-22.04 ]
    runs-on: ${{ matrix.os }}
    concurrency:
      group: build-${{ github.workflow }}-${{ matrix.os }}-${{ github.event.number || github.ref }}
      cancel-in-progress: true
    steps:
      - name: Checkout the repo
        uses: actions/checkout@v4.2.2
        with:
          fetch-depth: 0
      - uses: nicolasfara/build-check-deploy-sbt-action@1.0.14
        with:
          should-run-codecov: ${{ runner.os == 'Linux' }}
          codecov-directory: "target/"
          check-command: sbt +scalafmtCheckAll +scalafmtSbtCheck '+scalafixAll --check' +test +coverageAggregate
          should-deploy: false
          maven-central-username: ${{ secrets.MAVEN_CENTRAL_USERNAME }}
          maven-central-password: ${{ secrets.MAVEN_CENTRAL_PASSWORD }}
          signing-key: ${{ secrets.SIGNING_KEY }}
          signing-password: ${{ secrets.SIGNING_PASSWORD }}
          codecov-token: ${{ secrets.CODECOV_TOKEN }}
  release:
    permissions:
      contents: write
      packages: write
    concurrency:
      # Only one release job at a time. Strictly sequential.
      group: release-${{ github.workflow }}-${{ github.event.number || github.ref }}
    needs:
      - build
    runs-on: ubuntu-latest
    if: >-
      !github.event.repository.fork
      && (
        github.event_name != 'pull_request'
        || github.event.pull_request.head.repo.full_name == github.repository
      )
    steps:
      - name: Checkout
        uses: actions/checkout@v4.2.2
        with:
          token: ${{ secrets.DEPLOYMENT_TOKEN }}
      - name: Install Node
        uses: actions/setup-node@v4.2.0
        with:
          node-version-file: package.json
      - uses: nicolasfara/build-check-deploy-sbt-action@1.0.13
        with:
          build-command: true
          check-command: true
          deploy-command: |
            npm install
            npx semantic-release
          retries-on-failure: 1
          should-run-codecov: false
          should-deploy: true
          github-token: ${{ github.token }}
          maven-central-username: ${{ secrets.MAVEN_CENTRAL_USERNAME }}
          maven-central-password: ${{ secrets.MAVEN_CENTRAL_PASSWORD }}
          signing-key: ${{ secrets.SIGNING_KEY }}
          signing-password: ${{ secrets.SIGNING_PASSWORD }}
```


## Contributing

Contributions are welcome! Please open an issue or submit a pull request with your changes.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
