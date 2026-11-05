# ods-jenkins-shared-library Build Guide

This guide explains how to build **ods-jenkins-shared-library** (the OpenDevStack Jenkins Pipeline shared library) on any machine: Linux, macOS (Intel or Apple Silicon), or Windows, on x86_64 or ARM64.

Unless stated otherwise, paths are relative to the repository root (the folder that contains this file). Replace placeholders such as `<path-to-jdk11>` with locations of your choice.

---

## 1. What you are building

| Item | Value |
|------|-------|
| Project | Jenkins Pipeline shared library: global steps in `vars/` (29 files), classes in `src/` (package `org.ods...`), Spock unit tests in `test/groovy`, test fixtures in `test/resources` |
| Groovy | **2.4.21** (`org.codehaus.groovy:groovy-all:2.4.21` declared in `build.gradle`; resolved version confirmed with `gradlew dependencyInsight`). Also `groovy-cps 1.29`, `jenkins-core 2.46.3`, Spock `1.3-groovy-2.4`, JenkinsPipelineUnit `1.9`, CodeNarc `1.6`, JaCoCo `0.8.6` |
| Build tool | Gradle **7.4.2** through the included wrapper (`gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.jar`, `gradle/wrapper/gradle-wrapper.properties`) |
| Required JDK | **JDK 11** (Java toolchain `languageVersion = 11` in `build.gradle`; the project CI in `.github/workflows/gradle.yml` uses Temurin 11; `.devcontainer` uses Java 11). Verified: Gradle running on JDK 11 (build successful), and Gradle running on JDK 17 with JDK 11 auto-provisioned as toolchain (build successful, in Docker). Gradle 7.4.2 **cannot run on JDK 21** (verified failure). |
| Output | `build/libs/<folder-name>-0.0.1-SNAPSHOT.jar` (about 1.9 MB, compiled classes; not used by Jenkins, see section 6), test reports in `build/reports/tests/test/`, CodeNarc reports in `build/reports/codenarc/` |

The jar is only a by-product. Jenkins loads a shared library directly from the Git repository and compiles the Groovy sources itself, so the purpose of this build is to check that the sources compile, pass the CodeNarc checks and pass the unit tests, exactly like the project CI (`./gradlew clean build` with `NO_NEXUS=true`).

---

## 2. Things to know before building

### 2.1 JDK 11 toolchain

The build requires a JDK 11 toolchain. Two ways to satisfy it:

- **Recommended:** run Gradle itself on JDK 11 (`JAVA_HOME` pointing to a JDK 11). This is what the CI does.
- Run Gradle on JDK 12 to 17 and let Gradle find a JDK 11 installed on the machine, or download one automatically (toolchain auto-provisioning is enabled by default). Verified in the `eclipse-temurin:17-jdk` container: Gradle 7.4.2 downloaded JDK 11 from `https://api.adoptopenjdk.net/...` into `~/.gradle/jdks/`. This needs internet access to that host; auto-provisioning on macOS ARM64 or Windows was not tested.

Gradle 7.4.2 does not run on JDK 18+ officially; on JDK 21 it fails with `Unsupported class file major version 65` (verified). A JRE is not enough: the toolchain must be a full JDK (with `javac`), otherwise Gradle reports `does not provide the required capabilities: [JAVA_COMPILER]`.

### 2.2 Nexus or public repositories

`build.gradle` chooses the repositories with the `no_nexus` property, which **defaults to `true`**, so a normal build uses public repositories and needs no credentials:

- Maven Central (`https://repo.maven.apache.org/maven2/` and `https://repo1.maven.org/maven2/`)
- Jenkins releases `https://repo.jenkins-ci.org/releases/`
- `mavenLocal()` (your `~/.m2/repository`, if present)
- `https://services.gradle.org` (Gradle 7.4.2 distribution downloaded by the wrapper)

These must be reachable (whitelist them on a proxy or firewall). Inside an OpenDevStack installation the build can use a Nexus instead, with `NO_NEXUS=false` plus `NEXUS_URL` (or `NEXUS_HOST`), `NEXUS_USERNAME`, `NEXUS_PASSWORD` (or the Gradle properties `no_nexus`, `nexus_url`, `nexus_user`, `nexus_pw`). The Nexus mode was not tested.

### 2.3 CodeNarc can fail the build

`build` runs `codenarcMain`, which fails if there is any priority 1 or 2 violation or more than **300** priority 3 violations. At this revision the report shows 295 priority 3 violations, so the build passes, but new code can push it over the limit. `codenarcTest` never fails the build (`ignoreFailures = true`) and prints `CodeNarc rule violations were found. See the report at: ...build/reports/codenarc/test.html`; that line is expected.

### 2.4 Very verbose test output

The `test` task sets `showStandardStreams = true`, so the console shows a lot of pipeline log text, including lines such as `WARN: Error with Aqua due to: java.lang.NullPointerException ...` and `Not executed - Disabled at project level`. These come from tests that check error handling and are expected. Judge the result by `BUILD SUCCESSFUL` (test failures do fail this build) and by `build/reports/tests/test/index.html`.

### 2.5 Jar name depends on the folder name

There is no `settings.gradle`, so the project name (and the jar name) is the name of the folder that contains the repository. In a folder named `ods-jenkins-shared-library` the jar is `ods-jenkins-shared-library-0.0.1-SNAPSHOT.jar`; in the Docker command of section 5 (mounted as `/src`) it is `src-0.0.1-SNAPSHOT.jar`.

---

## 3. Install JDK 11

Any OpenJDK 11 build works (Eclipse Temurin, Azul Zulu, Amazon Corretto, Microsoft, Liberica, etc.). Native ARM64 builds of JDK 11 exist for Linux and for Apple Silicon (for example Temurin 11 and Zulu 11). Install the **JDK**, not a JRE.

| Platform | Command |
|----------|---------|
| SDKMAN (Linux, macOS, WSL) | `sdk install java 11.<x>-tem` (pick a version from `sdk list java`) |
| Debian / Ubuntu | `sudo apt install openjdk-11-jdk` |
| Fedora / RHEL | `sudo dnf install java-11-openjdk-devel` |
| Arch | `sudo pacman -S jdk11-openjdk` |
| macOS (Homebrew) | `brew install --cask temurin@11` |
| Windows (winget) | `winget install EclipseAdoptium.Temurin.11.JDK` |
| Windows (Chocolatey) | `choco install temurin11` |
| Manual | https://adoptium.net/temurin/releases/?version=11 |

Point `JAVA_HOME` at it.

**Linux / macOS / WSL:**

```bash
export JAVA_HOME=<path-to-jdk11>
export PATH="$JAVA_HOME/bin:$PATH"
```

**Windows PowerShell:**

```powershell
$env:JAVA_HOME = "<path-to-jdk11>"
$env:Path = "$env:JAVA_HOME\bin;$env:Path"
```

**Windows cmd.exe:**

```bat
set JAVA_HOME=<path-to-jdk11>
set PATH=%JAVA_HOME%\bin;%PATH%
```

Verify:

```bash
java -version     # must print version "11.x"
javac -version    # must exist (full JDK)
```

No separate Gradle installation is needed: the wrapper downloads Gradle 7.4.2 on first use.

---

## 4. Build

From the repository root:

```bash
./gradlew --no-daemon clean build          # Linux / macOS / WSL
gradlew.bat --no-daemon clean build        # Windows
```

`build` compiles `src/` and `vars/`, builds the jar, runs CodeNarc on main and test sources, compiles and runs the Spock unit tests. Measured on an x86_64 Linux machine: about 2 to 3 minutes (1m 46s with dependencies cached, 2m 52s in a fresh container that downloaded everything). Expected end of the output:

```
> Task :test
> Task :check
> Task :build

BUILD SUCCESSFUL
```

Verified result at this revision: 6207 tests, 0 failures, 0 errors, 0 skipped (counted from `build/test-results/test/*.xml`).

Outputs:

| Path | Description |
|------|-------------|
| `build/libs/<folder-name>-0.0.1-SNAPSHOT.jar` | Compiled classes of `src/` and `vars/` (section 2.5) |
| `build/reports/tests/test/index.html` | HTML test report |
| `build/test-results/test/*.xml` | JUnit XML results |
| `build/reports/codenarc/main.html`, `test.html` | CodeNarc reports |

Other useful commands (verified on Linux; `codenarcMain` was verified as part of `build`, not on its own):

```bash
./gradlew --no-daemon clean test jacocoTestReport   # what "make test" runs; coverage in build/reports/jacoco/
./gradlew --no-daemon groovydoc                     # API docs in build/docs/groovydoc/
./gradlew --no-daemon compileGroovy                 # compile the library sources only
./gradlew --no-daemon codenarcMain                  # style checks only
./gradlew --no-daemon javaToolchains                # show the JDKs Gradle detected
```

The CI also passes `-Djava.io.tmpdir=<some-temp-dir>`; it is optional.

On Windows use `gradlew.bat` instead of `./gradlew`. The Windows commands were not run during verification (only Linux was available). The `make doc` target additionally needs Go (`go run render-adoc.go`) and was not tested.

---

## 5. Build with Docker (no local JDK)

Works on any OS that runs Docker. From the repository root:

```bash
docker run --rm --user "$(id -u):$(id -g)" -e HOME=/tmp \
  -v "$PWD":/src -w /src eclipse-temurin:17-jdk \
  ./gradlew --no-daemon -Dorg.gradle.jvmargs=-Xmx1g build
```

Verified on Linux x86_64 with a 3 GB container memory limit: `BUILD SUCCESSFUL in 2m 52s`, 6207 tests passed. Inside the container Gradle runs on JDK 17 and automatically downloads a JDK 11 for the toolchain (section 2.1), so the container needs internet access to `api.adoptopenjdk.net` as well as the repositories in section 2.2. The jar is written as `build/libs/src-0.0.1-SNAPSHOT.jar` (section 2.5), owned by your user. The `-Dorg.gradle.jvmargs=-Xmx1g` flag keeps the Gradle process small; the test JVM uses its own 2 GB heap (`maxHeapSize = "2048m"`), so give Docker at least 3 GB of memory.

Notes:

- On Windows PowerShell drop the `--user` option and replace `"$PWD":/src` with `"${PWD}:/src"` (not verified).
- On ARM64 (Apple Silicon, ARM servers) the `eclipse-temurin` image runs natively, but the toolchain download for `aarch64` was not verified. If it fails, use an `eclipse-temurin:11-jdk` image instead (then Gradle runs directly on JDK 11 and nothing is downloaded); that variant was not run during verification either.

---

## 6. Use the library in Jenkins

Jenkins does not use the jar. It checks out the library from Git and compiles it at pipeline run time.

1. In Jenkins, go to **Manage Jenkins > System > Global Pipeline Libraries** and add a library:
   - Name: `ods-jenkins-shared-library` (the name used in the documentation)
   - Default version: a branch such as `4.x` or a tag
   - Retrieval method: Modern SCM, Git, project repository `https://github.com/opendevstack/ods-jenkins-shared-library.git` (or your fork)
2. In a `Jenkinsfile`, load it and call one of the global steps from `vars/`, for example:

```groovy
@Library('ods-jenkins-shared-library@4.x') _

odsComponentPipeline(
  imageStreamTag: 'ods/jenkins-agent-golang:4.x',
  branchToEnvironmentMapping: ['master': 'dev']
) { context ->
  odsComponentStageBuildOpenShiftImage(context)
}
```

The library targets OpenDevStack on OpenShift (agents, Nexus, SonarQube, Bitbucket, Jira and so on). Within OpenDevStack the Jenkins master is preconfigured with the library, and the comment in `build.gradle` explains that new dependencies must also be preloaded there (`ods-core`, `init.groovy.d/ods-jenkins-shared-library.groovy`). See `docs/modules/jenkins-shared-library/pages/` (published at https://www.opendevstack.org/ods-documentation/). Running the library in a live Jenkins was not part of this verification.

---

## 7. Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `Could not compile build file '.../build.gradle'` with `Unsupported class file major version 65` | Gradle 7.4.2 is running on JDK 21. Use JDK 11 (or 17) (section 2.1). |
| `Toolchain installation '<path>' does not provide the required capabilities: [JAVA_COMPILER]` | A JRE was found where a JDK is needed. Install the full JDK 11 (section 3). |
| `No compatible toolchains found for request filter: {languageVersion=11...}` or a failed download from `api.adoptopenjdk.net` | Gradle runs on another JDK and cannot find or download JDK 11. Install JDK 11 and run Gradle on it. |
| `Nexus URL not specified!` | `NO_NEXUS`/`no_nexus` is set to `false` without `NEXUS_URL`. Unset it (default is public repositories) or provide the Nexus settings (section 2.2). |
| `Could not resolve org.jenkins-ci...` / `Could not GET 'https://repo.jenkins-ci.org/...'` | The Jenkins repository is blocked (section 2.2). |
| `Execution failed for task ':codenarcMain'` ... `exceeded the maximum` | Too many CodeNarc violations (section 2.3). Fix them; see `build/reports/codenarc/main.html`. |
| `CodeNarc rule violations were found ... test.html` | Expected, `codenarcTest` ignores failures (section 2.3). |
| Lots of `WARN: Error with Aqua ... NullPointerException` in the test output | Expected test log output (section 2.4). |
| Jar called `src-0.0.1-SNAPSHOT.jar` or another unexpected name | The jar takes the folder name (section 2.5). |
| Out of memory | The test JVM needs about 2 GB on top of Gradle. Make sure the machine or container has at least 3 GB available. |

---

## 8. Quick reference

```bash
# Requirement: JDK 11 (full JDK)
java -version                               # 11.x

# Build + CodeNarc + unit tests (CI-equivalent)
./gradlew --no-daemon clean build           # -> build/libs/<folder-name>-0.0.1-SNAPSHOT.jar
./gradlew --no-daemon clean test jacocoTestReport   # tests + coverage

# Docker, no local JDK (JDK 11 toolchain downloaded automatically)
docker run --rm --user "$(id -u):$(id -g)" -e HOME=/tmp -v "$PWD":/src -w /src eclipse-temurin:17-jdk ./gradlew --no-daemon -Dorg.gradle.jvmargs=-Xmx1g build

# Jenkins: Global Pipeline Library "ods-jenkins-shared-library", then in a Jenkinsfile:
# @Library('ods-jenkins-shared-library@4.x') _
```
