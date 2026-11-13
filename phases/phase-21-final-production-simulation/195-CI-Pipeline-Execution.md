# PHASE 21 — FINAL PRODUCTION SIMULATION

# CHAPTER 195 — CI PIPELINE EXECUTION

Filename: `195-CI-Pipeline-Execution.md`

This chapter continues from Chapter 194. The Order Tracking API PR has been reviewed, approved and merged into `main`.

Now we will execute the Continuous Integration (CI) process for the exact merged source revision.

Our goal: Understand how an enterprise CI pipeline receives source code, builds it, tests it, checks code quality, handles credentials, records evidence and produces a validated build that can proceed to security scanning and Docker image creation.

This chapter focuses on the CI execution lifecycle. The detailed security scanning workflow and Docker image build are covered separately in Chapters 196 and 197.

# 1. CONTINUOUS PRODUCTION SCENARIO

We are working on the same enterprise e-commerce platform.

| Item                | Value                             |
| ------------------- | --------------------------------- |
| Jira issue          | `ECOM-4821`                       |
| Feature             | Order Tracking API                |
| Microservice        | `order-service`                   |
| Repository          | `company-ecommerce/order-service` |
| Source branch       | `main`                            |
| Merged commit       | `d4e5f6a` (illustrative)          |
| Language            | Java 21                           |
| Framework           | Spring Boot                       |
| Build tool          | Maven                             |
| CI platform         | GitHub Actions                    |
| Code quality        | SonarQube                         |
| Security tools      | Trivy, Veracode                   |
| Container registry  | Amazon ECR                        |
| Kubernetes          | Amazon EKS                        |
| Deployment approach | Helm + ArgoCD                     |

The commit SHA is illustrative. In a real implementation, always use the full SHA returned by GitHub.

Our workflow so far:

```
193 Developer Feature Workflow
          |
          v
194 Pull Request Workflow
          |
          v
PR Approved and Merged
          |
          v
main branch updated
          |
          v
195 CI Pipeline Execution  <-- WE ARE HERE
          |
          v
196 Security Scanning Pipeline
          |
          v
197 Docker Image Build
          |
          v
198 ECR Image Publishing
```

## 1.1 What does CI mean?

Continuous Integration is the practice of frequently integrating code changes into a shared repository and automatically validating those changes.

When developers merge changes into `main`, the CI system can automatically:

1. Detect the new commit.
2. Start a pipeline.
3. Check out the source code.
4. Set up the build environment.
5. Download dependencies.
6. Compile the application.
7. Run tests.
8. Generate test reports.
9. Perform code-quality analysis.
10. Package the application.
11. Record the build result.
12. Pass the validated source and build evidence to the next stages.

CI is not simply running `mvn clean install`.

A production CI pipeline also needs reliable execution, secure permissions, traceability, failure handling, repeatability and auditability.

## 1.2 CI versus CD

| CI                            | CD                                         |
| ----------------------------- | ------------------------------------------ |
| Integrates and validates code | Delivers or deploys validated software     |
| Compiles application          | Promotes artifacts                         |
| Runs tests                    | Updates environment configuration          |
| Performs code-quality checks  | Deploys through approved release processes |
| Produces build evidence       | Tracks release and deployment state        |

In our project, CI validates the Order Service code. Later chapters cover artifact publication, GitOps promotion and deployment.

# 2. UNDERSTAND THE COMPLETE CI ARCHITECTURE

## 2.1 High-level architecture

[CI/CD Pipeline for Java App Using GitHub Actions and AWS EKS | by Charan Reddy | Medium](https://images.openai.com/static-rsc-4/NXYFNgLrXwdC3BtyPkmLjE-ScQLKDg6hVnISfWVQLRyMp9o_M2JfTuEd6jTXMmQmRsHNYs4vFHuBx_KQxmIPp0HiCpLiwA1c-p_8JsCIOtVqujZFGPIcBnDbNVg7aC7EfBPtlStw8tKQCSb7zdjVLgX64K66zejD7QOCwYvlf4o?purpose=inline)

[medium.com](https://medium.com/%40charangajulapalli2001/ci-cd-pipeline-for-java-app-using-github-actions-and-aws-eks-2077fc383ea4?utm_source=chatgpt.com)

The logical flow is:

```
Developer
   |
   v
GitHub Repository
   |
   | Push event to main
   v
GitHub Actions
   |
   v
CI Runner
   |
   +---- Checkout Source
   |
   +---- Setup Java 21
   |
   +---- Maven Dependency Resolution
   |
   +---- Compile
   |
   +---- Unit Tests
   |
   +---- Integration Tests
   |
   +---- Package JAR
   |
   +---- SonarQube Analysis
   |
   +---- Quality Gate
   |
   v
CI Result and Build Evidence
   |
   v
Chapter 196 — Security Scanning
```

This diagram represents the logical workflow. Some stages may execute in parallel or in separate jobs in the actual implementation.

## 2.2 Components involved

| Component                | Responsibility                                            |
| ------------------------ | --------------------------------------------------------- |
| GitHub                   | Stores source code and sends repository events            |
| GitHub Actions           | Orchestrates workflow execution                           |
| GitHub-hosted runner     | Provides an execution environment                         |
| Self-hosted runner       | Provides company-managed execution infrastructure         |
| Git                      | Retrieves source code and commit information              |
| Java 21                  | Application build runtime                                 |
| Maven                    | Dependency management, compilation, testing and packaging |
| SonarQube                | Code quality and static analysis                          |
| GitHub Actions artifacts | Stores selected build outputs and reports                 |
| AWS IAM                  | Controls access to AWS resources when needed              |

## 2.3 Where does the runner execute?

A CI runner is the machine or execution environment that performs the workflow commands.

There are two common options.

GitHub-hosted runner

GitHub provisions and manages the runner environment.

Example:

```
runs-on: ubuntu-latest
```

Advantages:

- No need to manage runner operating systems.
- Easy to scale workflow execution.
- Suitable for many standard builds.
- Less infrastructure maintenance.

Limitations:

- Environment is managed by GitHub.
- Network access to private infrastructure may need additional design.
- Usage limits and billing depend on the plan and runner type.
- Ephemeral environments should not be treated as persistent build servers.

Self-hosted runner

The organization manages its own runner infrastructure.

Example:

```
runs-on: [self-hosted, linux, x64, order-service]
```

Advantages:

- More control over the environment.
- Access to private networks when securely configured.
- Can use preinstalled build dependencies.
- Can support specialized hardware or tools.

Limitations:

- The company owns maintenance and patching.
- Runner isolation becomes a security responsibility.
- Persistent workspaces can cause cross-build contamination.
- Poorly secured runners can expose credentials and internal systems.

## 2.4 Which runner should we use?

For this simulation, use a GitHub-hosted Linux runner for the standard Java build.

```
runs-on: ubuntu-latest
```

This avoids introducing unnecessary runner-management complexity into the example.

If a production organization requires private network access, special build dependencies or controlled execution infrastructure, the Platform team can provide isolated self-hosted runners.

Do not automatically place every application pipeline on a shared, permanently running EC2 machine.

# 3. UNDERSTAND THE REPOSITORY STRUCTURE

Our application repository:

```
company-ecommerce/
└── order-service/
    ├── .github/
    │   └── workflows/
    │       ├── pr-validation.yml
    │       └── ci-main.yml
    │
    ├── src/
    │   ├── main/
    │   │   ├── java/
    │   │   └── resources/
    │   │
    │   └── test/
    │       ├── java/
    │       └── resources/
    │
    ├── helm/
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── templates/
    │
    ├── Dockerfile
    ├── pom.xml
    ├── mvnw
    ├── mvnw.cmd
    ├── .mvn/
    ├── .gitignore
    └── README.md
```

### 3.1 Why separate PR and main workflows?

We use separate workflow files for clarity.

`pr-validation.yml` handles validation when a PR is opened or updated.

`ci-main.yml` handles the merged main-branch source.

They may share reusable workflow components, but their triggers and permissions should reflect their different responsibilities.

### 3.2 Why use Maven Wrapper?

The Maven Wrapper consists of repository-managed files that let developers and CI use a consistent Maven version without depending on whatever Maven happens to be installed on the machine.

Commands:

```
./mvnw --version
```

Example:

```
Apache Maven 3.9.x
Java version: 21
```

The actual version depends on the wrapper configuration.

Using `./mvnw` instead of relying on a globally installed `mvn` helps make builds more consistent.

# 4. UNDERSTAND THE MAIN-BRANCH TRIGGER

## 4.1 What triggers the CI pipeline?

A GitHub Actions workflow can start when a push occurs to the `main` branch.

For example:

```
PR #4821 merged
       |
       v
GitHub updates main
       |
       v
Push event generated
       |
       v
GitHub Actions evaluates workflow triggers
       |
       v
CI workflow starts
```

The PR merge and the main-branch push are related events, but they are not the same workflow trigger.

The PR workflow validates proposed changes. The main workflow validates the resulting main-branch revision.

## 4.2 Create the main CI workflow

File:

```
.github/workflows/ci-main.yml
```

Complete illustrative baseline:

```
name: Order Service Main CI

on:
  push:
    branches:
      - main

  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: order-service-main-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-and-test:
    name: Build and Test
    runs-on: ubuntu-latest

    timeout-minutes: 30

    steps:
      - name: Checkout source
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Display build environment
        run: |
          java -version
          ./mvnw --version

      - name: Validate Maven project
        run: ./mvnw validate

      - name: Compile application
        run: ./mvnw compile

      - name: Run unit tests
        run: ./mvnw test

      - name: Package application
        run: ./mvnw package

      - name: Run complete Maven verification
        run: ./mvnw verify

      - name: Upload build outputs and test reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: order-service-ci-results
          path: |
            target/*.jar
            target/surefire-reports/
            target/failsafe-reports/
          if-no-files-found: warn
          retention-days: 14
```

This is a learning baseline, not a complete hardened enterprise pipeline. Later sections explain how to improve it and remove unnecessary repeated Maven executions.

For a hardened production workflow, pin third-party actions to approved immutable commit SHAs and use the organization's action allowlist.

## 4.3 Understand each workflow section

### Workflow name

```
name: Order Service Main CI
```

This is the name displayed in GitHub Actions.

### Push trigger

```
on:
  push:
    branches:
      - main
```

This starts the workflow when a push event updates `main`.

### Manual trigger

```
workflow_dispatch:
```

This allows an authorized user to start the workflow manually from GitHub.

A manual trigger should still follow the same validation and authorization policies.

### Permissions

```
permissions:
  contents: read
```

The workflow only needs repository read access for this build.

It does not need permission to change repository files, publish releases or modify PRs.

### Concurrency

```
concurrency:
  group: order-service-main-${{ github.ref }}
  cancel-in-progress: true
```

This groups runs for the same branch.

If a newer main-branch run starts, an older in-progress run may be cancelled.

Use this only when cancelling an older run is safe. If every main commit must be processed, do not blindly cancel previous runs; use an appropriate queueing strategy.

### Timeout

```
timeout-minutes: 30
```

Prevents a job from running indefinitely.

Choose the timeout based on actual build duration and expected variation.

### Checkout

```
uses: actions/checkout@v4
```

Downloads the repository source into the runner workspace.

### Java setup

```
uses: actions/setup-java@v4
```

Installs the specified Java distribution and version.

### Maven caching

```
cache: maven
```

Caches Maven dependencies to reduce repeated downloads.

The cache is an optimization. A correct build must still work when the cache is empty.

### Build and test

```
run: ./mvnw verify
```

Runs the Maven verification lifecycle.

We will examine the Maven lifecycle in detail shortly.

### Upload artifacts

The workflow stores selected outputs and reports so that engineers can inspect the build after it finishes.

Artifacts are not the same thing as a production release or a container image published to ECR.

# 5. WHAT HAPPENS INSIDE THE RUNNER?

## 5.1 Runner execution lifecycle

When GitHub starts the job:

```
GitHub Actions receives event
         |
         v
Job is scheduled
         |
         v
Runner allocated
         |
         v
Workspace prepared
         |
         v
Workflow steps executed
         |
         v
Logs and results recorded
         |
         v
Job completes
```

On a GitHub-hosted runner, the environment is generally provisioned for the job and discarded afterward.

Do not assume that files from one run are automatically available to another run.

Use explicit artifacts or approved caches when data needs to cross job boundaries.

## 5.2 Inspect the workspace

Inside a runner step:

```
pwd
```

Displays the current working directory.

```
ls -la
```

Lists files.

```
git status
```

Shows the checked-out repository status.

```
git rev-parse HEAD
```

Displays the exact checked-out commit SHA.

```
git branch --show-current
```

Displays the current branch where applicable. In detached checkout scenarios, the branch output may not be the expected branch name; the SHA is the reliable source identity.

## 5.3 Add a source identity step

For traceability, add:

```
- name: Record source revision
  run: |
    echo "Repository: $GITHUB_REPOSITORY"
    echo "Commit: $(git rev-parse HEAD)"
    echo "Event: $GITHUB_EVENT_NAME"
    echo "Ref: $GITHUB_REF"
```

This helps connect the CI run to the exact source revision.

For our example:

```
Repository: company-ecommerce/order-service
Commit: d4e5f6a...
Event: push
Ref: refs/heads/main
```

The full SHA must be captured in the actual execution.

# PART 2 — MAVEN BUILD LIFECYCLE

# 6. UNDERSTAND MAVEN BEFORE RUNNING THE PIPELINE

Maven is the build and dependency management tool used by our Java Spring Boot application.

It handles:

- Dependency resolution.
- Source compilation.
- Test compilation.
- Unit test execution.
- Integration test execution when configured.
- Packaging.
- Verification.
- Artifact installation and deployment when requested.

The Maven configuration is stored in:

```
pom.xml
```

## 6.1 Understand the Maven lifecycle

Maven has three built-in lifecycles:

1. `clean`
2. `default`
3. `site`

The default lifecycle contains phases such as:

```
validate
   |
   v
compile
   |
   v
test
   |
   v
package
   |
   v
verify
   |
   v
install
   |
   v
deploy
```

These are lifecycle phases, not necessarily separate independent builds.

When Maven runs a later phase, it executes the preceding phases in that lifecycle.

For example:

```
./mvnw package
```

Executes the applicable preceding phases, including compilation and tests.

Similarly:

```
./mvnw verify
```

Executes the earlier default lifecycle phases and the verification phase.

Important: Do not run `validate`, `compile`, `test`, `package` and `verify` as separate sequential CI commands without a reason. Each command may repeat work. In the final optimized pipeline, one `clean verify` execution is generally more efficient.

## 6.2 What does `clean` do?

```
./mvnw clean
```

Removes the previous build output, normally the `target/` directory.

Why is this useful?

Suppose an earlier build generated:

```
target/order-service.jar
```

A later build fails before producing a new JAR.

Without proper cleanup, old outputs can create confusion during troubleshooting or packaging.

A clean build helps prevent stale generated files from being mistaken for current outputs.

## 6.3 What does `validate` do?

```
./mvnw validate
```

Checks whether the project is correctly configured and ready for the build.

Examples of things that can fail:

- Invalid Maven project configuration.
- Missing required plugin configuration.
- Incorrect project metadata.
- Invalid POM structure.

## 6.4 What does `compile` do?

```
./mvnw compile
```

Compiles the application's main source code.

Example:

```
src/main/java/
       |
       v
Java compiler
       |
       v
target/classes/
```

If a developer introduces a Java compilation error, this stage fails.

Example:

```
String status = order.getStatus(
```

The incomplete statement causes compilation failure.

## 6.5 What does `test` do?

```
./mvnw test
```

Compiles and executes the configured unit tests.

Example source:

```
src/test/java/
```

A typical Spring Boot project uses JUnit and may use Mockito for unit-test mocking.

Example output:

```
Tests run: 35
Failures: 0
Errors: 0
Skipped: 0
```

The actual count depends on the project.

## 6.6 What does `package` do?

```
./mvnw package
```

Packages the compiled application into its configured artifact format.

For our Spring Boot application, the expected output is typically an executable JAR:

```
target/order-service-1.0.0.jar
```

The filename depends on the Maven project configuration.

## 6.7 What does `verify` do?

```
./mvnw verify
```

Runs the lifecycle through the verification phase.

This can include:

- Unit tests.
- Integration tests, if correctly configured.
- Additional verification plugins.
- Packaging checks.
- Project-specific quality validations.

Do not assume integration tests run simply because the command says `verify`. The project must configure integration-test execution, for example through Maven Failsafe or another test framework.

## 6.8 What does `install` do?

```
./mvnw install
```

Installs the generated artifact into the local Maven repository, usually:

```
~/.m2/repository/
```

This is useful when another local Maven project needs the artifact.

It is not normally necessary just to build a deployable Spring Boot application in CI.

## 6.9 What does `deploy` do?

```
./mvnw deploy
```

Publishes the Maven artifact to a configured remote Maven repository.

This is not the same as deploying the application to Kubernetes.

In our container-based delivery flow, the application will eventually be packaged into a Docker image and published to ECR.

Do not confuse Maven artifact deployment with production application deployment.

## 6.10 Maven lifecycle reference

| Command           | Purpose                                         |
| ----------------- | ----------------------------------------------- |
| `./mvnw clean`    | Remove previous build output                    |
| `./mvnw validate` | Validate project setup                          |
| `./mvnw compile`  | Compile application source                      |
| `./mvnw test`     | Run configured unit tests                       |
| `./mvnw package`  | Create application package                      |
| `./mvnw verify`   | Run lifecycle verification                      |
| `./mvnw install`  | Install artifact locally                        |
| `./mvnw deploy`   | Publish artifact to configured Maven repository |

# 7. UNDERSTAND THE ORDER SERVICE POM.XML

The Maven configuration is important because CI depends on it.

Example simplified `pom.xml` for our Spring Boot service:

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
         http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.company.ecommerce</groupId>
    <artifactId>order-service</artifactId>
    <version>1.0.0</version>

    <packaging>jar</packaging>

    <properties>
        <java.version>21</java.version>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.0</version>
        <relativePath/>
    </parent>

    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <build>
        <plugins>

            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>

        </plugins>
    </build>

</project>
```

This is a teaching example, not a verified copy of the actual Order Service POM. Use the project's existing dependency versions, parent, plugins and Java configuration. Do not replace a working production POM blindly.

## 7.1 Why define the Java version?

```
<maven.compiler.release>21</maven.compiler.release>
```

This ensures the compiler targets the intended Java release.

The runner's Java version and the project's configured Java version should be compatible.

A mismatch can cause:

- Compilation errors.
- Unsupported class file versions.
- Different behavior between local and CI builds.

## 7.2 Why use a Spring Boot parent?

The Spring Boot parent provides managed dependency and plugin defaults.

It can reduce the need to manually specify versions for every Spring dependency.

In a production project, dependency versions should be managed consistently and upgraded through a controlled process.

## 7.3 Why use Maven Wrapper?

The repository includes:

```
mvnw
mvnw.cmd
.mvn/wrapper/
```

The wrapper configuration controls the Maven distribution version.

The developer and CI should use the same wrapper.

Check:

```
cat .mvn/wrapper/maven-wrapper.properties
```

The exact content depends on the wrapper configuration.

# PART 3 — BUILD AND TEST EXECUTION

# 8. DEPENDENCY RESOLUTION

Before compilation, Maven must resolve the project's dependencies.

For example:

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

Maven resolves the required libraries from configured repositories.

The common public repository is Maven Central. Organizations may also use internal artifact repositories or mirrors.

The dependency resolution flow:

```
pom.xml
   |
   v
Maven reads dependencies
   |
   v
Checks local Maven cache
   |
   +---- Dependency available?
   |             |
   |          Yes|----> Use cached dependency
   |             |
   |            No
   |             |
   |             v
   |      Download from repository
   |             |
   |             v
   |      Store in local cache
   |
   v
Continue build
```

## 8.1 Where are Maven dependencies stored?

Normally:

```
~/.m2/repository/
```

Inside a Linux runner:

```
ls -la ~/.m2/repository
```

This directory may be cached between workflow executions through the GitHub Actions cache.

## 8.2 Why does dependency resolution fail?

Possible causes:

- Maven Central or the internal repository is unavailable.
- Network restrictions.
- Incorrect proxy configuration.
- Invalid repository credentials.
- Dependency version does not exist.
- TLS or certificate problems.
- Repository rate limiting.

Example:

```
Could not resolve dependencies
Could not transfer artifact
Connection timed out
```

Troubleshooting:

```
./mvnw -U dependency:resolve
```

`-U` tells Maven to check for updated snapshots and missing releases according to Maven's resolution behavior.

Inspect:

```
cat ~/.m2/settings.xml
```

Be careful: this file can contain credentials. Do not print or upload secrets to public CI logs.

For enterprise repositories, credentials should be supplied through an approved secret mechanism.

## 8.3 Dependency integrity and reproducibility

A production build should avoid relying on uncontrolled dependency changes.

Recommended practices:

- Use explicit dependency versions or a managed BOM.
- Review dependency upgrades.
- Use approved repositories.
- Maintain dependency locks or verification controls where supported by the build ecosystem.
- Record dependency and build metadata.
- Scan dependencies in the designated security stage.
- Avoid downloading arbitrary binaries from untrusted URLs.

Maven dependency caching must not be confused with dependency verification. A cache improves speed; it does not prove that a dependency is trustworthy.

# 9. COMPILE THE APPLICATION

The pipeline compiles Java source code.

Command:

```
./mvnw -B -ntp compile
```

Options:

- `-B`: batch mode, suitable for CI.
- `-ntp`: suppresses transfer progress output to keep logs cleaner.
- `compile`: executes the compilation lifecycle phase.

Example:

```
[INFO] Compiling 85 source files
[INFO] BUILD SUCCESS
```

The number of files is illustrative.

## 9.1 Where are compiled files stored?

```
target/classes/
```

Inspect:

```
find target/classes -type f | head
```

Compiled Java `.class` files appear here.

## 9.2 Compilation failure example

```
[ERROR] cannot find symbol
symbol: method getTrackingStatus()
location: class Order
```

Possible reasons:

- Incorrect method name.
- Missing dependency.
- Model changed without updating callers.
- Incorrect import.
- Source code inconsistency.

Correct the source and rerun the build.

Do not skip compilation because a previous local build passed.

# 10. UNIT TEST EXECUTION

Unit tests validate individual application components.

For example:

- `OrderTrackingService`
- `OrderTrackingController`
- `OrderRepository` interactions through mocks
- Response mapping
- Error handling

Example test:

```
@ExtendWith(MockitoExtension.class)
class OrderTrackingServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderTrackingService orderTrackingService;

    @Test
    void shouldReturnTrackingForCustomerOrder() {

        Order order = new Order();

        order.setOrderId("ORD-1001");
        order.setCustomerId("CUS-1001");
        order.setStatus("SHIPPED");

        when(orderRepository
                .findByOrderIdAndCustomerId(
                        "ORD-1001",
                        "CUS-1001"))
                .thenReturn(Optional.of(order));

        OrderTrackingResponse response =
                orderTrackingService.getTracking(
                        "ORD-1001",
                        "CUS-1001");

        assertEquals("ORD-1001", response.orderId());
        assertEquals("SHIPPED", response.status());
    }
}
```

This example assumes that the corresponding entity, repository and service contracts exist. Adapt it to the real project.

Run:

```
./mvnw -B -ntp test
```

## 10.1 What does Mockito do?

Mockito allows tests to create mock dependencies.

In this example:

```
@Mock
private OrderRepository orderRepository;
```

A mock repository is used instead of a real database.

This keeps the service unit test focused on service behavior.

## 10.2 Why unit tests are important in CI

Imagine a developer changes order status handling.

Previously:

```
SHIPPED
```

New implementation accidentally returns:

```
UNKNOWN
```

A suitable unit test should detect the regression.

Without automated tests, the defect might reach an environment and be discovered only by QA or customers.

## 10.3 What are test reports?

Maven Surefire commonly produces reports under:

```
target/surefire-reports/
```

Example:

```
target/surefire-reports/
├── TEST-com.company.OrderTrackingServiceTest.xml
└── com.company.OrderTrackingServiceTest.txt
```

These reports provide test execution details.

Read a text report:

```
cat target/surefire-reports/*.txt
```

Example:

```
Tests run: 10, Failures: 0, Errors: 0, Skipped: 0
```

The actual report depends on the test framework and plugin configuration.

## 10.4 Test failure example

```
Tests run: 10
Failures: 1
Errors: 0
Skipped: 0
```

The pipeline must fail if the required test execution reports a failure.

Do not use:

```
./mvnw test || true
```

This hides failures and makes a broken build appear successful.

# 11. INTEGRATION TEST EXECUTION

Unit tests and integration tests serve different purposes.

Unit test: Tests one component in isolation.

Integration test: Tests interactions between multiple components or real infrastructure dependencies.

For the Order Service, integration tests may cover:

- Spring application context.
- REST controller and service interaction.
- Database access.
- Repository behavior.
- Authentication and authorization integration.
- External service adapters.

The exact dependencies are determined by the actual service design. We will not assume a particular database or messaging platform for this feature.

## 11.1 Maven Failsafe

A common way to manage integration tests is the Maven Failsafe Plugin.

Example:

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.5.2</version>

    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

This is an illustrative plugin configuration. Verify plugin versions and project compatibility before adopting it.

Integration tests can follow a naming convention such as:

```
OrderTrackingIntegrationTest.java
```

## 11.2 Why use Failsafe instead of only Surefire?

Surefire is commonly used for unit tests.

Failsafe is designed for integration tests and separates integration-test execution from the final verification outcome.

This can help manage test setup and teardown more safely.

## 11.3 What does the CI runner need for integration tests?

It depends on what the application integrates with.

For example, if the test requires a database, the workflow needs an isolated test database or a suitable ephemeral test service.

If the test requires an external API, use a controlled test endpoint or mock server.

Never point CI integration tests at production databases or live customer systems.

## 11.4 Example CI test reports

```
target/
├── surefire-reports/
└── failsafe-reports/
```

The pipeline should retain the reports when tests fail, because they help engineers identify the root cause.

## 11.5 Integration test failure example

```
Connection refused
```

Possible reasons:

- Test database did not start.
- Wrong connection string.
- Service port not available.
- Test container startup failed.
- Application configuration is incorrect.

Troubleshooting should start with the test infrastructure logs, not by randomly increasing test timeouts.

# 12. PACKAGE THE APPLICATION

After tests pass, Maven packages the application.

Command:

```
./mvnw -B -ntp package
```

For a typical Spring Boot executable JAR:

```
target/
└── order-service-1.0.0.jar
```

Verify:

```
ls -lh target/*.jar
```

Inspect the JAR:

```
jar tf target/order-service-1.0.0.jar | head
```

Run the JAR locally where the required configuration is available:

```
java -jar target/order-service-1.0.0.jar
```

This command is for local validation, not an instruction to start a long-running application inside the CI job.

## 12.1 Why retain the JAR?

The JAR is a build output.

It can be used to:

- Inspect packaging.
- Validate the application artifact.
- Support container creation.
- Provide build evidence.
- Diagnose packaging problems.

For our delivery process, Chapter 197 will use the application package when constructing the Docker image.

# PART 4 — CODE QUALITY WITH SONARQUBE

# 13. WHY SONARQUBE IS PART OF CI

A successful compilation does not mean the source code is high quality.

Consider this example:

```
public Order getOrder(String id) {

    try {
        return orderRepository.findById(id).get();
    } catch (Exception e) {
        return null;
    }
}
```

This code might compile, but it has several potential problems:

- Calling `get()` without checking whether a value exists.
- Catching a broad exception.
- Returning `null` without a clear contract.
- Hiding the actual error.
- Making troubleshooting harder.

SonarQube can identify configured code-quality and security-related issues.

## 13.1 What does SonarQube analyze?

Depending on edition and configuration, analysis may include:

- Bugs.
- Vulnerabilities.
- Code smells.
- Security hotspots.
- Duplicated code.
- Test coverage.
- Maintainability.
- Reliability.
- Complexity.

SonarQube does not replace code review, penetration testing or all application security testing.

## 13.2 SonarQube architecture

[docs.sonarsource.com](https://docs.sonarsource.com/sonarqube-server/2025.3/analyzing-source-code/analysis-overview?utm_source=chatgpt.com)

Logical flow:

```
GitHub Actions Runner
        |
        v
Maven Build and Tests
        |
        v
Sonar Scanner
        |
        v
SonarQube Server
        |
        v
Source Analysis
        |
        v
Quality Gate Evaluation
        |
        v
Pass / Fail
```

The scanner runs in the CI environment and submits analysis data to the SonarQube server.

The server evaluates the analysis against the configured quality profile and quality gate.

## 13.3 Where is SonarQube installed?

A company may operate SonarQube as:

- A dedicated VM or EC2-based service.
- A managed or internally hosted platform.
- A containerized service.
- A platform shared by multiple development teams.

For our project, assume the organization already provides an approved internal SonarQube service.

We will not create another SonarQube installation in every microservice repository.

A centralized service is easier to govern and maintain.

## 13.4 Configure project properties

Create:

```
sonar-project.properties
```

Example:

```
sonar.projectKey=company-order-service
sonar.projectName=Order Service

sonar.sources=src/main/java
sonar.tests=src/test/java

sonar.java.binaries=target/classes

sonar.junit.reportPaths=target/surefire-reports

sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml

sonar.sourceEncoding=UTF-8
```

This is an example configuration. Paths must match the actual Maven project and its test and coverage plugin setup.

For Maven-based projects, configuration may also be supplied through Maven properties rather than this file.

## 13.5 Configure JaCoCo coverage

JaCoCo is a Java code coverage tool.

It generates information about which parts of the code are exercised by tests.

Example Maven configuration:

```
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>

    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>

        <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

This configuration is illustrative; use a plugin version compatible with the project's Java and Maven setup.

Run:

```
./mvnw clean verify
```

Expected report location:

```
target/site/jacoco/index.html
target/site/jacoco/jacoco.xml
```

The XML report can be consumed by SonarQube.

## 13.6 Integrate SonarQube with GitHub Actions

The SonarQube server URL and token must be supplied securely.

Example repository configuration:

```
Repository Settings
       |
       v
Secrets and variables
       |
       v
Actions
       |
       +---- SONAR_HOST_URL
       |
       +---- SONAR_TOKEN
```

`SONAR_HOST_URL` can be a non-secret variable if organizational policy allows it.

`SONAR_TOKEN` must be a protected secret.

Add a SonarQube step to the workflow after test and coverage report generation:

```
- name: SonarQube analysis
  env:
    SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  run: |
    ./mvnw -B -ntp \
      org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
      -Dsonar.host.url="$SONAR_HOST_URL"
```

For a production pipeline, pin the Sonar Maven scanner plugin version approved by the Platform team rather than relying on an unspecified version.

Do not print the token.

## 13.7 What is a SonarQube quality gate?

A quality gate is a set of conditions that determines whether the analyzed code meets the organization's quality requirements.

For example, an organization might require:

- No new blocker issues.
- No new critical vulnerabilities.
- Minimum new-code coverage.
- Limited duplicated code.
- No unacceptable security findings.

The actual thresholds must be set by the organization.

Avoid assuming that a particular coverage percentage or vulnerability count is universally correct.

## 13.8 How does CI know whether the quality gate passed?

The scanner submits the analysis. SonarQube evaluates it and returns a quality-gate result.

A pipeline must wait for and enforce the result if the quality gate is intended to block progression.

Do not assume that starting the scanner automatically means the quality gate has passed.

Depending on the integration, use the scanner's supported quality-gate waiting mechanism or a separate quality-gate check.

Example Maven invocation with a waiting option:

```
./mvnw -B -ntp \
  org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
  -Dsonar.qualitygate.wait=true
```

Use this with the approved scanner version and server configuration.

## 13.9 SonarQube failure example

```
Quality Gate Status: FAILED

New code coverage: below configured threshold
```

Correct approach:

1. Open the SonarQube project.
2. Inspect the new-code analysis.
3. Identify untested code paths.
4. Add meaningful tests.
5. Rerun Maven verification.
6. Submit new analysis.
7. Confirm the updated quality gate.

Do not add meaningless tests just to increase a coverage number.

## 13.10 SonarQube troubleshooting

| Problem                          | Possible cause                       | Action                            |
| -------------------------------- | ------------------------------------ | --------------------------------- |
| Scanner cannot connect           | Network or server unavailable        | Verify URL and connectivity       |
| Authentication failed            | Invalid token                        | Rotate and update secret          |
| Project not found                | Incorrect project key or permissions | Check project configuration       |
| No coverage detected             | JaCoCo report missing                | Verify report generation and path |
| Analysis succeeds but gate fails | Quality conditions unmet             | Inspect new-code findings         |
| Scanner version error            | Plugin incompatibility               | Use approved compatible version   |

# PART 5 — GITHUB ACTIONS CI ARCHITECTURE

# 14. SINGLE JOB VERSUS MULTIPLE JOBS

A small project may use a single CI job.

A larger enterprise application may separate work into jobs.

Example:

```
                  CI Workflow
                      |
                      v
               Checkout Source
                      |
                      v
                 Build Job
                      |
                      v
                Unit Test Job
                      |
                      v
             Integration Test Job
                      |
                      v
              SonarQube Analysis
                      |
                      v
               Quality Gate
                      |
                      v
              CI Success Result
```

This is a logical illustration. Jobs may execute in parallel when dependencies and artifact handling permit.

## 14.1 Why split jobs?

Advantages:

- Easier ownership.
- Better visibility.
- Different runner requirements.
- Parallel execution.
- Independent retry possibilities.
- Clearer troubleshooting.

Disadvantages:

- More workflow complexity.
- Artifact transfer between jobs.
- Additional runner startup time.
- Possible repeated dependency downloads.
- More configuration to maintain.

Do not split every Maven phase into a separate job without a measurable reason.

## 14.2 Understanding `needs`

GitHub Actions uses `needs` to establish job dependencies.

Example:

```
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Build completed"

  unit-tests:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Unit tests completed"

  integration-tests:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Integration tests completed"
```

Here, the two test jobs wait for the build job.

However, each job has its own runner and filesystem. The `build` job's files do not automatically appear in the other jobs.

If a later job needs an output, use an explicit artifact transfer or rebuild the required content.

## 14.3 Upload and download artifacts between jobs

Build job:

```
- name: Upload compiled application
  uses: actions/upload-artifact@v4
  with:
    name: order-service-jar
    path: target/*.jar
    if-no-files-found: error
```

Another job:

```
- name: Download compiled application
  uses: actions/download-artifact@v4
  with:
    name: order-service-jar
    path: downloaded/
```

This is an example of artifact transfer.

For the final production design, the team must decide whether jobs share one build or independently rebuild the application.

The important principle is to know which artifact is being tested and eventually released.

# 15. FINAL OPTIMIZED CI WORKFLOW

The earlier baseline demonstrated individual Maven phases. Running them all separately repeats work.

Here is a more efficient single-job workflow for our example.

File:

```
.github/workflows/ci-main.yml
```

```
name: Order Service Main CI

on:
  push:
    branches:
      - main

  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: order-service-main-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-test-quality:
    name: Build, Test and Quality
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - name: Checkout source
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Record source revision
        run: |
          echo "Repository: $GITHUB_REPOSITORY"
          echo "Commit: $(git rev-parse HEAD)"
          echo "Event: $GITHUB_EVENT_NAME"
          echo "Ref: $GITHUB_REF"

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Verify build environment
        run: |
          java -version
          ./mvnw --version

      - name: Run clean verification
        run: ./mvnw -B -ntp clean verify

      - name: Upload test reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: order-service-test-reports
          path: |
            target/surefire-reports/
            target/failsafe-reports/
          if-no-files-found: warn
          retention-days: 14

      - name: Upload application package
        if: success()
        uses: actions/upload-artifact@v4
        with:
          name: order-service-package
          path: target/*.jar
          if-no-files-found: error
          retention-days: 14

      - name: SonarQube analysis
        if: success()
        env:
          SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          ./mvnw -B -ntp \
            org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
            -Dsonar.host.url="$SONAR_HOST_URL" \
            -Dsonar.qualitygate.wait=true
```

This is a compact example. Before production use:

- Pin actions to approved immutable SHAs.
- Pin the Sonar scanner plugin version.
- Confirm the JaCoCo report is generated during `verify`.
- Ensure SonarQube has the correct project configuration.
- Configure the server and token at repository or organization level.
- Confirm quality-gate waiting is supported by the selected scanner setup.
- Use an appropriate concurrency strategy for your release policy.
- Validate integration-test setup.
- Ensure the uploaded JAR is the exact artifact later used by the Docker build.

Important design note: If Maven verification and Sonar analysis run as separate Maven invocations, they may repeat lifecycle work. A production implementation can configure the scanner in the same Maven invocation or use a carefully designed analysis job with transferred reports. The goal is one authoritative source revision and one controlled build artifact, not unnecessary repeated builds.

# PART 6 — CI SECRETS, IAM AND PERMISSIONS

# 16. WHY CI NEEDS SECURE CREDENTIAL MANAGEMENT

Our Order Service pipeline may need credentials to connect to external systems such as:

- SonarQube.
- Private Maven repositories.
- Approved package registries.
- AWS services in later stages.

These credentials must not be stored directly in source code or workflow YAML.

## 16.1 What is a GitHub Actions secret?

A GitHub Actions secret is an encrypted value that can be made available to authorized workflows.

Examples:

```
SONAR_TOKEN
MAVEN_REPOSITORY_USERNAME
MAVEN_REPOSITORY_PASSWORD
```

Do not create secrets unless the corresponding integration needs them.

## 16.2 Create a repository secret

Navigate to:

```
GitHub Repository
       |
       v
Settings
       |
       v
Secrets and variables
       |
       v
Actions
       |
       v
New repository secret
```

Example:

```
Name:
SONAR_TOKEN

Value:
<actual-secret-issued-by-sonarqube>
```

The value above is a placeholder. Never commit a real token into GitHub.

## 16.3 Access the secret in a workflow

```
- name: SonarQube analysis
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  run: |
    ./mvnw -B -ntp \
      org.sonarsource.scanner.maven:sonar-maven-plugin:sonar
```

The secret is exposed to that step through an environment variable.

Avoid:

```
- run: echo "${{ secrets.SONAR_TOKEN }}"
```

Never print credentials into logs.

Also avoid placing secret values directly into command strings where they may appear in process arguments or diagnostic output.

## 16.4 Repository secrets versus organization secrets

| Repository secret                            | Organization secret                        |
| -------------------------------------------- | ------------------------------------------ |
| Belongs to one repository                    | Managed centrally                          |
| Suitable for repository-specific credentials | Suitable for approved shared integrations  |
| Managed by repository administrators         | Managed by organization administrators     |
| Limited to configured repository access      | Can be restricted to selected repositories |

For a company with 21 microservices, centralized secret management can reduce duplicated administration.

However, shared secrets should not give all services access to the same broad privileges.

Use separate credentials or narrowly scoped access wherever possible.

## 16.5 What should never be stored in Git?

Examples:

```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
DATABASE_PASSWORD
SONAR_TOKEN
JWT_PRIVATE_KEY
MAVEN_REPOSITORY_PASSWORD
```

Also protect:

- Private SSH keys.
- Signing keys.
- TLS private keys.
- Cloud credentials.
- API tokens.
- Kubernetes credentials.

A `.gitignore` file can prevent accidental local files from being added:

```
.env
.env.*
!.env.example

*.pem
*.key

settings-local.xml
credentials.json

target/
```

`.gitignore` is not a security boundary. It does not prevent someone from intentionally adding a secret or stop a previously committed secret from remaining in history.

## 16.6 Use GitHub OIDC for AWS access

In later chapters, the workflow may need to access AWS for image publication or other approved operations.

Prefer short-lived identity federation through GitHub Actions OpenID Connect (OIDC), where supported, rather than storing long-lived AWS access keys in GitHub secrets.

Logical flow:

```
GitHub Actions Workflow
         |
         v
Requests OIDC token
         |
         v
AWS IAM Identity Provider
         |
         v
IAM Role Trust Policy
         |
         v
Temporary AWS Credentials
         |
         v
Authorized AWS Operations
```

The workflow must be restricted to the intended repository, branch or protected environment.

Example IAM trust-policy conditions should verify claims such as:

- Repository identity.
- Organization identity.
- Branch or environment identity.
- Intended audience.

Do not use an unrestricted trust policy.

A later chapter will define the permissions required for ECR publishing. The CI build job itself does not need ECR write permission merely to compile Java code.

## 16.7 Least privilege for CI

For our basic build:

```
permissions:
  contents: read
```

This is preferable to granting every permission by default.

If a later job needs to create a GitHub release or update a repository, grant only the permissions required by that job.

Do not give the entire CI workflow write access because one future stage might need it.

# PART 7 — CI PERFORMANCE AND CACHING

# 17. WHY PIPELINE PERFORMANCE MATTERS

Suppose every Order Service build takes 25 minutes.

If developers push multiple changes each day, they spend a significant amount of time waiting for feedback.

A slow CI pipeline affects:

- Developer productivity.
- Release frequency.
- Incident fix turnaround.
- Engineering cost.
- CI runner consumption.

Our goal is to make builds efficient without reducing validation quality.

## 17.1 Measure the baseline first

Suppose the initial pipeline takes:

| Stage                       | Example duration |
| --------------------------- | ---------------- |
| Runner startup and checkout | 1 min            |
| Dependency downloads        | 6 min            |
| Compilation                 | 3 min            |
| Unit tests                  | 5 min            |
| Integration tests           | 7 min            |
| SonarQube                   | 3 min            |
| Total                       | 25 min           |

These are illustrative values, not measurements from the user's actual project.

Do not optimize based on guesses. Capture actual stage durations from GitHub Actions.

## 17.2 Maven dependency caching

Without caching:

```
Build 1:
Download dependencies
Compile
Test
Package

Build 2:
Download dependencies again
Compile
Test
Package
```

With caching:

```
Build 1:
Download dependencies
Save cache

Build 2:
Restore cache
Download only missing dependencies
Compile
Test
Package
```

GitHub Actions setup-java supports Maven caching:

```
- name: Set up Java
  uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: '21'
    cache: maven
```

This typically caches Maven dependency data based on the dependency configuration.

The first build may still be slow because the cache must be populated.

## 17.3 Do not cache the entire target directory blindly

Avoid treating:

```
target/
```

as a universal dependency cache.

It contains generated build outputs that may become stale or environment-specific.

Cache reusable dependencies, not arbitrary generated outputs.

## 17.4 Parallel test execution

If the test suite supports safe parallel execution, it may reduce build time.

However, parallel tests can fail when they share:

- Database records.
- Ports.
- File locations.
- External test accounts.
- Static mutable state.
- Shared test infrastructure.

Before enabling parallelism, ensure test isolation.

Do not parallelize tests simply because it is possible.

## 17.5 Split independent jobs

For example:

```
Build
  |
  +---- Unit Tests
  |
  +---- Independent Static Checks
  |
  +---- Documentation Validation
```

Only independent jobs should execute concurrently.

If the SonarQube job requires a coverage report, it must wait for the job producing that report or receive it through an artifact.

## 17.6 Avoid unnecessary Maven executions

Bad pattern:

```
- run: ./mvnw clean
- run: ./mvnw validate
- run: ./mvnw compile
- run: ./mvnw test
- run: ./mvnw package
- run: ./mvnw verify
```

This may repeat lifecycle phases.

Better baseline:

```
- run: ./mvnw -B -ntp clean verify
```

Add separate invocations only when there is a concrete reason.

## 17.7 Use concurrency carefully

Example:

```
concurrency:
  group: order-service-main-${{ github.ref }}
  cancel-in-progress: true
```

This is useful when a newer commit makes an older build unnecessary.

But consider this case:

```
Commit A pushed
Commit B pushed
Commit C pushed
```

If the team requires every commit to be validated or recorded, cancelling earlier runs may violate the workflow.

Choose a concurrency strategy that matches the release and audit requirements.

## 17.8 Example performance optimization plan

| Problem                         | Optimization                                         |
| ------------------------------- | ---------------------------------------------------- |
| Repeated dependency downloads   | Maven cache                                          |
| Repeated Maven phases           | Run one complete verification lifecycle              |
| Slow tests                      | Profile slow tests and improve isolation             |
| Serial independent checks       | Parallelize where safe                               |
| Slow runner provisioning        | Use appropriate runner strategy                      |
| Large unnecessary build context | Keep Docker work out of CI build stage unless needed |
| Repeated source checkout        | Use reusable jobs and artifacts thoughtfully         |

Do not remove integration tests or security controls simply to meet a build-time target.

# PART 8 — JENKINS, GITHUB ACTIONS AND GITLAB CI/CD

# 18. UNDERSTAND THE THREE CI PLATFORMS

Our project documentation includes Jenkins, GitHub Actions and GitLab CI/CD.

They all support automation, but they are not identical products.

| Capability                 | Jenkins                | GitHub Actions                          | GitLab CI/CD                                      |
| -------------------------- | ---------------------- | --------------------------------------- | ------------------------------------------------- |
| Pipeline definition        | Jenkinsfile            | Workflow YAML                           | `.gitlab-ci.yml`                                  |
| Execution                  | Jenkins agents         | GitHub-hosted or self-hosted runners    | GitLab runners                                    |
| Source control integration | Plugins and webhooks   | Native GitHub integration               | Native GitLab integration                         |
| Credentials                | Jenkins Credentials    | Actions secrets / OIDC                  | CI/CD variables and supported identity federation |
| Pipeline reuse             | Shared Libraries       | Reusable workflows and actions          | Includes and components                           |
| Typical administration     | Jenkins administrators | GitHub administrators and Platform team | GitLab administrators and Platform team           |

## 18.1 What is Jenkins?

Jenkins is an automation server used to build, test and deliver software.

A Jenkins installation commonly has:

```
Jenkins Controller
       |
       +---- Agent 1
       |
       +---- Agent 2
       |
       +---- Agent 3
```

The controller schedules jobs and agents execute work.

In our historical migration example, Jenkins was hosted on EC2 with multiple build agents.

Jenkins remains useful in organizations that have substantial existing automation, custom plugins or specialized execution requirements.

## 18.2 What is GitHub Actions?

GitHub Actions is GitHub's workflow automation platform.

Example repository path:

```
.github/workflows/ci-main.yml
```

It integrates with GitHub events and can execute workflows using hosted or self-hosted runners.

## 18.3 What is GitLab CI/CD?

GitLab CI/CD uses a YAML pipeline definition, commonly:

```
.gitlab-ci.yml
```

It uses GitLab Runners to execute jobs.

Example:

```
stages:
  - build
  - test

build:
  stage: build
  image: maven:3.9-eclipse-temurin-21
  script:
    - mvn -B -ntp clean compile

unit_tests:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  script:
    - mvn -B -ntp test
```

This is a basic illustration. A production GitLab pipeline should use approved image versions, dependency caching, proper artifacts, secure variables and appropriate runner permissions.

## 18.4 Why might an enterprise use more than one platform?

Possible reasons:

- Existing Jenkins pipelines have not yet been migrated.
- Different business units use different source-control platforms.
- A team standardizes on GitHub while another uses GitLab.
- Migration is performed incrementally.
- Some legacy applications depend on Jenkins plugins or infrastructure.

Using three platforms without a reason increases operational complexity.

Our project uses GitHub Actions as the primary workflow for the current scenario. Jenkins and GitLab are included for understanding enterprise alternatives and migration architecture.

## 18.5 Can one pipeline trigger another platform?

Yes, through controlled integration.

For example:

```
GitHub Repository
       |
       v
GitHub Actions
       |
       v
Approved API or webhook
       |
       v
External CI system
       |
       v
Pipeline execution
```

This requires:

- Authentication.
- Authorization.
- Event validation.
- Duplicate-event protection.
- Traceability.
- Failure handling.

Do not create an uncontrolled chain of pipelines that repeatedly trigger one another.

# PART 9 — CI OBSERVABILITY AND NOTIFICATIONS

# 19. HOW DO WE KNOW WHETHER CI IS HEALTHY?

A CI pipeline itself is a system that requires monitoring.

We should understand:

- Build success rate.
- Build failure rate.
- Average duration.
- Queue time.
- Runner availability.
- Test failure trends.
- Dependency download failures.
- Quality-gate failures.
- Pipeline cancellation rate.

For our project, the operational observability stack includes Prometheus, Grafana and ELK. GitHub Actions also provides native workflow logs and run history.

Do not assume that every GitHub-hosted runner automatically exposes Prometheus metrics to our monitoring stack.

## 19.1 CI logs

GitHub Actions records:

- Workflow runs.
- Job status.
- Step output.
- Error messages.
- Execution duration.

The first troubleshooting location is normally:

```
GitHub Repository
      |
      v
Actions
      |
      v
Select workflow run
      |
      v
Select failed job
      |
      v
Open failed step
```

## 19.2 CI metrics

A company can collect selected CI operational metrics using approved APIs, exporters or integration services.

Example metric concepts:

```
ci_workflow_runs_total
ci_workflow_failures_total
ci_workflow_duration_seconds
ci_runner_queue_duration_seconds
ci_runner_unavailable_total
```

These are proposed metric names for illustration, not built-in GitHub Actions metrics.

The collector must define how each metric is populated and how duplicates, missing data and API limits are handled.

## 19.3 Grafana dashboard

A CI dashboard could display:

| Panel              | Purpose                         |
| ------------------ | ------------------------------- |
| Build success rate | Overall pipeline reliability    |
| Failed runs        | Number of failed executions     |
| Average duration   | Build performance               |
| Longest jobs       | Identify bottlenecks            |
| Runner queue time  | Detect capacity problems        |
| Test failures      | Identify unstable code or tests |

The dashboard should distinguish infrastructure failures from application test failures.

## 19.4 Example alert definitions

If the company exports CI metrics into Prometheus, an alert rule could be:

```
groups:
  - name: ci-pipeline-alerts

    rules:
      - alert: CIWorkflowFailureRateHigh
        expr: |
          (
            sum(rate(ci_workflow_failures_total[15m]))
            /
            clamp_min(
              sum(rate(ci_workflow_runs_total[15m])),
              0.001
            )
          ) > 0.25
        for: 10m

        labels:
          severity: warning

        annotations:
          summary: CI workflow failure rate is high
          description: >
            More than 25 percent of observed CI
            runs are failing over the evaluation window.
```

This rule is illustrative. The metrics must actually exist, and the alert should be validated against real workload patterns before use.

A failure-rate alert is more useful when combined with a minimum run-volume requirement so that one failed run does not create misleading noise.

## 19.5 Notifications

A failed CI run may notify:

- Developer.
- Service owner.
- Platform team.
- Engineering team channel.

Example notification:

```
CI FAILURE

Service: order-service
Branch: main
Commit: d4e5f6a...
Run: <actual-run-id>

Failed stage: Maven verification

Action:
Inspect the failing test report and resolve
the issue before continuing release progression.
```

Notifications should contain useful diagnostic context but must not include secrets, private tokens or sensitive customer data.

# PART 10 — CI FAILURE SCENARIOS AND RECOVERY

# 20. FAILURE: WORKFLOW DID NOT START

Symptom: A PR was merged, but no main CI run appeared.

Investigate:

1. Was the target branch actually updated?
2. Does the workflow exist in the correct branch?
3. Is the trigger configured for `main`?
4. Is the workflow file valid?
5. Was the workflow disabled?
6. Did repository or organization policy prevent execution?

Useful commands:

```
git fetch origin
git log -1 --oneline origin/main
```

Check the latest main commit.

Then inspect the repository's Actions tab and workflow configuration.

Do not assume that a merged PR always triggers every workflow. Event filters and repository policies determine execution.

# 21. FAILURE: JAVA VERSION MISMATCH

Symptom:

```
Unsupported class file major version
```

or:

```
invalid target release: 21
```

Possible causes:

- Runner uses the wrong Java version.
- Maven compiler configuration is incorrect.
- A plugin does not support the configured Java version.
- A dependency has incompatible bytecode.

Troubleshooting:

```
java -version
./mvnw --version
```

Inspect:

```
<maven.compiler.release>21</maven.compiler.release>
```

Confirm that the runner, compiler plugin and dependencies support the intended Java release.

# 22. FAILURE: MAVEN DEPENDENCY DOWNLOAD FAILS

Symptom:

```
Could not transfer artifact
Connection timed out
```

Check:

- Repository availability.
- Network connectivity.
- Proxy settings.
- Repository credentials.
- Dependency coordinates.
- TLS certificate trust.

Retrying a transient network failure may be appropriate.

Repeatedly retrying an invalid dependency version will not fix the underlying problem.

# 23. FAILURE: UNIT TEST FAILURE

Symptom:

```
Tests run: 50
Failures: 3
```

Open the failed workflow step.

Inspect:

```
cat target/surefire-reports/*.txt
```

Determine whether the problem is:

- A real application defect.
- An incorrect test expectation.
- Test data collision.
- Shared-state interference.
- A timing issue.
- An environment dependency.

Do not make the workflow green by skipping the failing test without an approved reason.

# 24. FAILURE: INTEGRATION TEST ENVIRONMENT UNAVAILABLE

Symptom:

```
Connection refused
```

or:

```
Failed to start ApplicationContext
```

Check:

- Required service startup.
- Container health.
- Environment variables.
- Test configuration.
- Network connectivity.
- Port conflicts.
- Database initialization.

Use isolated test dependencies and collect their logs.

Do not solve infrastructure failures by pointing tests at production systems.

# 25. FAILURE: SONARQUBE CONNECTION ERROR

Symptom:

```
Not authorized
```

or:

```
Failed to connect to SonarQube
```

Check:

```
SONAR_HOST_URL
SONAR_TOKEN
```

Confirm:

- Correct URL.
- Valid token.
- Token has required project permissions.
- Server is reachable.
- Secret is available to this workflow event.
- Proxy and TLS configuration are correct.

Do not expose the token in the logs during troubleshooting.

# 26. FAILURE: QUALITY GATE FAILED

Symptom:

```
Quality Gate: FAILED
```

Possible causes:

- New code coverage below policy.
- New bugs.
- Vulnerability findings.
- Excessive duplication.
- Other configured quality conditions.

Action:

1. Open the SonarQube analysis.
2. Filter for new code.
3. Review the exact failed condition.
4. Fix the underlying issue.
5. Rerun the pipeline.
6. Verify the new quality-gate result.

# 27. FAILURE: ARTIFACT NOT FOUND

Symptom:

```
No files were found with the provided path
```

Possible causes:

- Maven packaging failed.
- Wrong artifact path.
- JAR filename differs.
- Packaging configuration changed.
- Upload step runs before the package exists.

Inspect:

```
find target -maxdepth 2 -type f
```

Then update the artifact path based on the actual build output.

For a required JAR, use:

```
if-no-files-found: error
```

rather than silently ignoring the missing package.

# 28. FAILURE: CI RUN CANCELLED

Symptom: A running job is cancelled when a newer run starts.

Check:

```
concurrency:
  group: order-service-main-${{ github.ref }}
  cancel-in-progress: true
```

Determine whether cancellation is expected.

If all main-branch revisions must complete, change the concurrency strategy.

Do not blindly disable concurrency or enable cancellation without considering the release policy.

# 29. FAILURE: BUILD PASSES LOCALLY BUT FAILS IN CI

This is one of the most common real-world problems.

Possible reasons:

- Different Java version.
- Different Maven version.
- Local environment variables missing in CI.
- Case-sensitive filesystem differences.
- Local dependency cache hides missing dependencies.
- Uncommitted local files.
- Different timezone or locale.
- Test depends on machine-specific state.
- Network or certificate differences.

Troubleshooting:

```
java -version
./mvnw --version
git status
./mvnw -B -ntp clean verify
```

Use the repository's Maven Wrapper and reproduce the CI command locally.

The goal is to make the build environment repeatable.

# PART 11 — BUILD REPRODUCIBILITY AND ARTIFACT TRACEABILITY

# 30. WHY MUST WE RECORD THE EXACT SOURCE COMMIT?

Imagine that the Order Service is deployed and customers report an issue with order tracking.

The operations team asks:

"Which source code version produced the running application?"

If the CI pipeline records only:

```
Branch: main
```

that is insufficient because `main` changes over time.

We need the exact source commit.

Example:

```
Repository:
company-ecommerce/order-service

Branch:
main

Source commit:
d4e5f6a...
```

This allows us to connect a build to a precise source revision.

## 30.1 Understand the CI run ID

Every GitHub Actions workflow run has a run ID.

Example:

```
Run ID: 9876543210
```

This is an illustrative identifier.

The run ID identifies the workflow execution, while the commit SHA identifies the source revision.

They are different.

| Identifier      | What it identifies    |
| --------------- | --------------------- |
| Jira issue      | Business requirement  |
| PR number       | Code review activity  |
| Commit SHA      | Exact source revision |
| Workflow run ID | CI execution          |
| Artifact name   | Stored build output   |
| Image digest    | Exact container image |

## 30.2 Record run metadata

Add this step to the workflow:

```
- name: Record build metadata
  run: |
    echo "Repository=$GITHUB_REPOSITORY"
    echo "Branch=$GITHUB_REF_NAME"
    echo "Commit=$(git rev-parse HEAD)"
    echo "Run ID=$GITHUB_RUN_ID"
    echo "Run Number=$GITHUB_RUN_NUMBER"
    echo "Actor=$GITHUB_ACTOR"
```

This records useful build context.

Do not use the actor name as proof that the actor personally executed every command. GitHub workflow execution may be automated or triggered through integrations.

## 30.3 Generate a build metadata file

Create a metadata file inside the runner:

```
- name: Generate build metadata
  run: |
    mkdir -p build-info

    cat > build-info/build-metadata.txt <<EOF
    SERVICE=order-service
    REPOSITORY=$GITHUB_REPOSITORY
    BRANCH=$GITHUB_REF_NAME
    COMMIT_SHA=$(git rev-parse HEAD)
    CI_RUN_ID=$GITHUB_RUN_ID
    CI_RUN_NUMBER=$GITHUB_RUN_NUMBER
    BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
    EOF
```

Upload it:

```
- name: Upload build metadata
  uses: actions/upload-artifact@v4
  with:
    name: order-service-build-metadata
    path: build-info/build-metadata.txt
    retention-days: 14
```

The retention period is an example. For production audit requirements, use an appropriate artifact repository and retention policy.

## 30.4 What is an artifact?

An artifact is a file produced by a build.

Examples:

```
order-service-1.0.0.jar
test-report.xml
coverage-report.xml
build-metadata.txt
```

Artifacts are different from:

- Git source code.
- Docker images.
- Helm charts.
- Kubernetes deployments.

A successful CI run does not automatically publish a container image.

## 30.5 Build once, promote the same artifact

This principle is critical for our later environment promotions.

Consider the wrong approach:

```
DEV:
Build JAR
Build Docker image

QA:
Build JAR again
Build Docker image again

STAGING:
Build JAR again
Build Docker image again

PROD:
Build JAR again
Build Docker image again
```

This can produce different artifacts even when the source branch appears unchanged.

Our intended approach:

```
Source Commit
      |
      v
One Controlled Build
      |
      v
One Application Package
      |
      v
One Docker Image
      |
      v
One ECR Image Digest
      |
      +------> DEV
      |
      +------> QA
      |
      +------> STAGING
      |
      +------> PROD
```

We build once and promote the same immutable image through environments.

The ECR digest will become the strongest artifact identity for our container delivery chain.

## 30.6 Reproducible build considerations

A reproducible build aims to produce the same output from the same source and controlled build inputs.

For stronger reproducibility:

- Pin Java and Maven versions.
- Pin build plugins.
- Control dependency versions.
- Use trusted repositories.
- Control build-time configuration.
- Avoid downloading unpinned tools.
- Record the source SHA.
- Record relevant build environment details.
- Avoid embedding unpredictable timestamps unless required.
- Record artifact checksums.

Not every ordinary Maven build is automatically bit-for-bit reproducible. A production organization should define and validate its reproducibility requirements.

## 30.7 Generate a SHA-256 checksum

For the JAR:

```
sha256sum target/*.jar
```

Example:

```
aabbccddeeff...  target/order-service-1.0.0.jar
```

The checksum can help detect changes to the file.

A checksum does not prove that the source is trustworthy. It proves whether the file matches a known checksum.

For stronger provenance, use an approved artifact attestation or signing solution, with verifiable identity and source information.

# PART 12 — MULTI-MICROSERVICE CI STRATEGY

# 31. WE HAVE 21 MICROSERVICES. HOW SHOULD WE MANAGE CI?

Our e-commerce platform has 21 microservices.

A poor design would place every microservice's build logic into one enormous workflow file.

Another poor design would duplicate hundreds of lines of CI configuration across all repositories.

We need reusable standards without losing service independence.

## 31.1 Recommended architecture

```
                 Platform Golden Path
                         |
                         |
             Reusable CI Workflow Templates
                         |
       +-----------------+-----------------+
       |                 |                 |
       v                 v                 v
  Order Service     Cart Service      Payment Service
       |                 |                 |
       v                 v                 v
  CI Workflow        CI Workflow        CI Workflow
       |                 |                 |
       v                 v                 v
  Build/Test         Build/Test         Build/Test
       |                 |                 |
       v                 v                 v
  CI Results         CI Results         CI Results
```

Each microservice should be independently buildable and testable.

## 31.2 Why use reusable workflows?

Reusable workflows allow a central Platform team to maintain common CI behavior.

For example:

```
platform-golden-path/
└── workflows/
    ├── java-ci.yml
    ├── node-ci.yml
    └── python-ci.yml
```

Application repositories call the approved reusable workflow.

This reduces duplication and makes standards easier to update.

## 31.3 Example reusable Java CI workflow

In a central repository:

```
platform-golden-path/.github/workflows/java-ci.yml
```

Example:

```
name: Reusable Java CI

on:
  workflow_call:
    inputs:
      java-version:
        required: false
        type: string
        default: '21'

      service-name:
        required: true
        type: string

    secrets:
      sonar-token:
        required: false

      sonar-host-url:
        required: false

jobs:
  build:
    runs-on: ubuntu-latest

    permissions:
      contents: read

    steps:
      - name: Checkout caller repository
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: ${{ inputs.java-version }}
          cache: maven

      - name: Build and test
        run: ./mvnw -B -ntp clean verify

      - name: Display service name
        run: echo "Service: ${{ inputs.service-name }}"
```

This is a minimal reusable workflow example.

A production implementation must also establish its action pinning, scanner integration, artifact handling, permissions and failure reporting standards.

## 31.4 Call the reusable workflow from Order Service

File:

```
order-service/.github/workflows/ci-main.yml
```

Example:

```
name: Order Service CI

on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  java-ci:
    uses: company-ecommerce/platform-golden-path/.github/workflows/java-ci.yml@v1

    with:
      java-version: '21'
      service-name: order-service
```

In a real organization, pin the reusable workflow reference to an approved immutable commit SHA or protected versioning mechanism.

Do not blindly trust a mutable branch for production workflow definitions.

## 31.5 Different microservices may use different technologies

For example:

| Microservice         | Technology         | Build approach |
| -------------------- | ------------------ | -------------- |
| Order Service        | Java / Spring Boot | Maven          |
| User Service         | Node.js            | npm            |
| Payment Service      | Java / Spring Boot | Maven          |
| Notification Service | Python             | pip/pytest     |
| Product Catalog      | Java / Spring Boot | Maven          |

These are illustrative technology assignments, not claims about the actual 21 service implementations.

The Platform team should maintain technology-specific reusable workflows.

Do not force a Node.js or Python service into a Java workflow.

## 31.6 Who owns the reusable workflow?

| Responsibility                  | Owner                                                       |
| ------------------------------- | ----------------------------------------------------------- |
| Workflow architecture           | Platform team                                               |
| Standard build templates        | Platform team                                               |
| Application-specific tests      | Service team                                                |
| Application dependency versions | Service team                                                |
| Security standards              | Security team                                               |
| Runner infrastructure           | Platform / CI operations                                    |
| Pipeline failure investigation  | Service team and Platform team, according to failure source |

The service team should own its application's correctness.

The Platform team should own the reliability and consistency of the shared pipeline platform.

# PART 13 — CI SECURITY AND PRODUCTION GOVERNANCE

# 32. PROTECT THE CI EXECUTION ENVIRONMENT

CI is part of the software supply chain.

A compromised CI runner can expose:

- Repository contents.
- Credentials available to the workflow.
- Build artifacts.
- Internal network access.
- Package publishing permissions.
- Cloud permissions.

Therefore, CI security must be designed intentionally.

## 32.1 Runner isolation

For self-hosted runners:

- Use isolated runner groups.
- Separate trusted and untrusted workflows.
- Avoid persistent workspaces across untrusted jobs.
- Patch operating systems and runner software.
- Restrict network access.
- Limit administrative access.
- Remove unused credentials.
- Monitor runner registration and execution.

Do not run untrusted PR code on a privileged production deployment runner.

## 32.2 Action supply-chain security

A GitHub Action is code that executes in the workflow.

For example:

```
uses: actions/checkout@v4
```

A production organization should:

- Review third-party actions.
- Use approved action sources.
- Pin actions to immutable commit SHAs.
- Track action updates.
- Restrict allowed actions through organization policy.
- Avoid unknown actions that request excessive permissions.

A version tag is convenient but can be moved. An immutable commit SHA provides a more stable reference.

## 32.3 Workflow permission boundaries

Separate jobs based on responsibility.

Example:

```
CI Build Job
    |
    | contents: read
    v
Compile and Test

Security Job
    |
    | Minimum required permissions
    v
Security Analysis

ECR Publishing Job
    |
    | OIDC + scoped AWS role
    v
Publish Image
```

The build job should not automatically receive the permissions of the ECR publishing job.

Use separate jobs and narrowly scoped credentials when appropriate.

## 32.4 Environment protection

GitHub environments can provide deployment-related protections, such as:

- Required reviewers.
- Deployment branch restrictions.
- Environment-specific secrets.
- Deployment history.

They are especially relevant when we reach production approval and deployment.

Do not expose production credentials to the general CI build job.

## 32.5 Auditability

Record:

- Who initiated a run.
- What event triggered it.
- Which repository was built.
- Which source SHA was checked out.
- Which workflow version executed.
- Which tools and versions were used.
- Which checks passed or failed.
- Which artifact was produced.
- Who authorized any privileged operation.

Keep audit information according to the organization's retention and compliance requirements.

# PART 14 — CI OPERATING PROCEDURES

# 33. WHAT SHOULD THE DEVOPS ENGINEER DO WHEN CI FAILS?

An engineer should not immediately rerun every failed pipeline.

First identify the failure category.

| Failure category            | Primary investigation                          |
| --------------------------- | ---------------------------------------------- |
| Workflow trigger            | GitHub event and workflow configuration        |
| Runner failure              | Runner availability and infrastructure         |
| Checkout failure            | Repository permissions and GitHub connectivity |
| Dependency failure          | Maven repository and network                   |
| Compilation failure         | Source code and Java configuration             |
| Unit test failure           | Test reports and application logic             |
| Integration test failure    | Test environment and external dependencies     |
| SonarQube failure           | Scanner connection and quality gate            |
| Artifact failure            | Package output and artifact paths              |
| Workflow permission failure | Token and permission configuration             |

## 33.1 Example incident: all microservice builds fail

Suppose the Order, Cart, Payment and User Service builds fail within the same 10-minute period.

Do not immediately assume that four development teams introduced defects at the same time.

Look for a shared dependency:

- Maven Central availability.
- Internal artifact repository.
- Shared runner outage.
- Organization-wide workflow template change.
- Shared SonarQube outage.
- Shared network or certificate issue.

The pattern of failures can indicate a platform-level incident.

## 33.2 Example incident: only Order Service fails

If all other Java services build successfully, but Order Service fails:

Investigate:

- Order Service source changes.
- Its POM configuration.
- Its tests.
- Service-specific dependencies.
- Its environment variables.
- Its repository workflow overrides.

The issue may be service-specific.

## 33.3 Avoid repeated blind retries

Retries are useful for transient failures.

They are not a substitute for root-cause analysis.

A useful operational approach:

```
Failure
   |
   v
Identify failed stage
   |
   v
Inspect logs
   |
   v
Classify failure
   |
   +---- Application issue
   |
   +---- Platform issue
   |
   +---- External dependency issue
   |
   +---- Configuration issue
   |
   v
Correct root cause
   |
   v
Rerun required validation
```

## 33.4 When should the DevOps engineer contact the developer?

Contact the developer when:

- Application compilation fails.
- Tests expose incorrect business logic.
- Application dependencies are incorrectly configured.
- Code quality issues need source changes.
- The feature behavior is incorrect.

Contact the Platform team when:

- Shared runners fail.
- Reusable workflows break.
- Shared secrets are unavailable.
- Organization-wide pipeline policy changes cause failures.
- Shared CI infrastructure is unavailable.

Contact the Security team when:

- A security policy blocks the pipeline.
- A credential exposure is suspected.
- A vulnerability requires an exception decision.
- A scanner or policy configuration needs security review.

# PART 15 — EXECUTE THE COMPLETE CI SCENARIO

# 34. END-TO-END EXECUTION

Now let's walk through the actual sequence that an engineer would follow for our Order Service.

## 34.1 Step 1 — Verify the merged source

On a local machine:

```
git checkout main
git pull --ff-only origin main
```

Check:

```
git log -1 --oneline
```

Example:

```
d4e5f6a feat(ECOM-4821): add order tracking API
```

## 34.2 Step 2 — Verify the source locally

```
java -version
./mvnw --version
./mvnw -B -ntp clean verify
```

Expected:

```
[INFO] Tests run: ...
[INFO] BUILD SUCCESS
```

Local validation does not replace CI. It helps detect problems before or during the pipeline execution.

## 34.3 Step 3 — Push or merge has already triggered CI

Our PR was already merged.

The main-branch workflow receives the push event and starts.

Navigate to:

```
GitHub
   |
   v
company-ecommerce/order-service
   |
   v
Actions
   |
   v
Order Service Main CI
```

## 34.4 Step 4 — Inspect the workflow run

The run should show:

```
Workflow:
Order Service Main CI

Branch:
main

Commit:
d4e5f6a...

Event:
push
```

Record the actual run ID from GitHub.

Example placeholder:

```
CI_RUN_ID=<actual-github-run-id>
```

## 34.5 Step 5 — Inspect each stage

Example successful execution:

```
Checkout source             SUCCESS
Record source revision      SUCCESS
Setup Java 21               SUCCESS
Maven dependency resolution SUCCESS
Maven compilation           SUCCESS
Unit tests                  SUCCESS
Integration tests           SUCCESS
Package application         SUCCESS
SonarQube analysis          SUCCESS
Quality gate                SUCCESS
Upload test reports         SUCCESS
Upload application package  SUCCESS
```

The stages shown here represent the intended completed workflow. The exact list depends on the workflow configuration.

## 34.6 Step 6 — Inspect build reports

Check:

```
target/surefire-reports/
target/failsafe-reports/
target/site/jacoco/
```

Review:

- Unit-test reports.
- Integration-test reports, if configured.
- Coverage report.
- SonarQube quality-gate result.

## 34.7 Step 7 — Inspect the generated artifact

Example:

```
order-service-1.0.0.jar
```

Check that the package exists and matches the expected service.

The actual filename must come from the Maven project configuration.

## 34.8 Step 8 — Record CI evidence

Example build record:

| Field              | Recorded value                    |
| ------------------ | --------------------------------- |
| Jira issue         | `ECOM-4821`                       |
| Repository         | `company-ecommerce/order-service` |
| Branch             | `main`                            |
| Source commit      | `d4e5f6a...`                      |
| PR                 | `#4821`                           |
| CI workflow        | `Order Service Main CI`           |
| CI run ID          | `<actual-run-id>`                 |
| Maven verification | Passed                            |
| Tests              | Passed                            |
| SonarQube          | Passed                            |
| Quality gate       | Passed                            |
| JAR artifact       | Generated                         |
| Docker image       | Not built yet                     |
| ECR image digest   | Not available yet                 |

The final three rows are important. We must not claim that an image has been built or published when we have only completed CI.

# PART 16 — FINAL CI VALIDATION CHECKLIST

## 35. Chapter 195 completion checklist

CI completion

# 0%

0 of 33 items verified

Main-branch CI trigger understood

CI architecture understood

Runner architecture understood

Repository structure understood

Maven Wrapper verified

Java 21 configured

Exact source commit recorded

GitHub workflow created

Workflow permissions restricted

Concurrency policy reviewed

Runner timeout configured

Maven dependency resolution understood

Maven lifecycle understood

Compilation validated

Unit tests executed

Integration test strategy documented

Test reports collected

Application package generated

SonarQube integration configured

JaCoCo coverage integration understood

Quality gate enforcement understood

CI secrets protected

AWS OIDC purpose understood

Dependency caching configured

Build optimization strategy understood

Jenkins/GitHub Actions/GitLab differences understood

Reusable workflow architecture understood

CI failure categories documented

CI troubleshooting procedures documented

Build metadata recorded

Artifact checksum and provenance understood

Security boundaries reviewed

Actual CI run ID recorded during execution

# PART 17 — INTERVIEW PREPARATION

## 36. Production-level CI interview questions

Q1. What happens when a developer merges a PR into main?

GitHub generates a push event for the target branch. The configured GitHub Actions workflow starts, checks out the source revision and executes the required build, test and quality checks.

Q2. What is the difference between a CI runner and a CI server?

A CI platform orchestrates pipeline execution. A runner is the execution environment where the job's commands actually run. In GitHub Actions, the workflow orchestrator schedules jobs on GitHub-hosted or self-hosted runners.

Q3. Why do we use Maven Wrapper?

It helps developers and CI use a consistent Maven version from the repository rather than relying on an arbitrary system installation.

Q4. Explain the Maven lifecycle.

Maven has `clean`, `default` and `site` lifecycles. The default lifecycle includes phases such as validate, compile, test, package, verify, install and deploy. Running a later phase executes the preceding phases in that lifecycle.

Q5. What is the difference between `package` and `verify`?

`package` creates the application package. `verify` executes the preceding default lifecycle phases and additional configured verification checks.

Q6. What is the difference between Surefire and Failsafe?

Surefire is commonly used for unit tests. Failsafe is commonly used for integration tests and supports their execution and verification through the Maven lifecycle.

Q7. How do you reduce Maven build time?

Use dependency caching, eliminate repeated lifecycle executions, optimize test execution, parallelize independent jobs where safe and investigate slow stages using actual pipeline measurements.

Q8. What is SonarQube?

SonarQube is a code-quality analysis platform. It analyzes code for configured bugs, vulnerabilities, code smells, duplication and coverage information, and can evaluate the result against a quality gate.

Q9. What is a quality gate?

A quality gate is a set of configured conditions that determines whether code analysis meets the organization's quality requirements.

Q10. How do you integrate SonarQube with GitHub Actions?

Configure the approved SonarQube server URL and token securely, execute the scanner during CI, submit the analysis and wait for or otherwise enforce the quality-gate result.

Q11. How do you manage CI credentials?

Use GitHub Actions secrets or approved external secret management. For AWS, prefer OIDC with a restricted IAM role and temporary credentials. Never commit credentials to source code.

Q12. What is the difference between GitHub-hosted and self-hosted runners?

GitHub-hosted runners are managed by GitHub. Self-hosted runners are managed by the organization, which takes responsibility for their security, maintenance, isolation and capacity.

Q13. How do you investigate a pipeline that suddenly becomes slow?

Compare stage durations against the baseline. Investigate runner queue time, dependency downloads, compilation, test execution and external analysis services. Optimize the actual bottleneck rather than removing necessary validation.

Q14. What happens if the build passes locally but fails in CI?

Compare Java and Maven versions, environment variables, dependency availability, filesystem behavior and test configuration. Reproduce the exact CI command using Maven Wrapper.

Q15. How do you manage CI for 21 microservices?

Use independently buildable service repositories with centrally maintained reusable workflows and technology-specific templates. Each service should retain ownership of its application code and tests.

Q16. Why should we record the commit SHA and CI run ID?

The commit SHA identifies the exact source revision, while the run ID identifies the execution that validated it. Both are necessary for traceability.

Q17. Why is an artifact checksum useful?

It helps verify that a file matches a previously recorded checksum. It does not independently prove that the file is trustworthy or authentic.

Q18. Why should we not give AWS permissions to every CI job?

Least privilege reduces the impact of compromised workflows. A Maven build does not need permission to publish production images or modify infrastructure.

Q19. How would you investigate failures across several microservices simultaneously?

Look for shared dependencies such as the runner platform, Maven repository, shared reusable workflow, network, certificates or SonarQube service before assuming that multiple teams introduced unrelated application defects.

Q20. What is the difference between CI artifact storage and ECR?

CI artifact storage can retain build outputs and reports. ECR is the container image registry that stores container images. They serve different purposes.

# PART 18 — CHAPTER 195 FINAL RESULT AND HANDOFF

## 37. What we have completed

We have covered the complete CI execution process for the Order Tracking feature:

- Main-branch trigger.
- GitHub Actions architecture.
- Runner types and execution.
- Maven configuration and lifecycle.
- Dependency resolution.
- Java compilation.
- Unit and integration testing.
- Test reports.
- Application packaging.
- SonarQube and JaCoCo.
- Quality-gate enforcement.
- GitHub Actions reusable workflows.
- Jenkins and GitLab CI/CD comparison.
- CI secrets and IAM permissions.
- OIDC architecture.
- Build caching and optimization.
- CI observability and notifications.
- CI failure scenarios.
- Build metadata and traceability.
- Multi-microservice CI design.
- CI security and governance.
- End-to-end execution and validation.

## 38. Current production simulation state

Order Service — ECOM-4821

Source branch

main

Source commit

d4e5f6a...

PR

\#4821

Maven build

Passed in simulation

Unit / integration tests

Passed in simulation

SonarQube quality gate

Passed in simulation

Application JAR

Generated in simulation

CI run ID\
Pending actual execution

Docker image

Not built

ECR image digest

Not published

These are simulated results for learning. No actual GitHub workflow was executed.

## 39. Next chapter

# CHAPTER 196 — SECURITY SCANNING PIPELINE

Filename: `196-Security-Scanning-Pipeline.md`

In the next chapter, we will take the same Order Service source revision and examine the security controls before the Docker image is published.

The chapter will cover:

1. DevSecOps security pipeline architecture.
2. Security scanning stages and execution order.
3. SonarQube security analysis and its role.
4. Trivy installation and configuration.
5. Dependency and filesystem scanning.
6. Secret scanning.
7. Docker image vulnerability scanning.
8. Veracode integration.
9. Security policies and severity thresholds.
10. Blocking versus non-blocking findings.
11. False positives and security exceptions.
12. Vulnerability remediation ownership.
13. CI credential security.
14. Security reports and audit evidence.
15. Failure scenarios and troubleshooting.
16. Complete security validation before image publishing.

Continuity: We will not change the Jira issue, service or source revision. We will continue from the validated `order-service` main-branch commit and preserve its build traceability. Docker image construction and ECR publishing remain in their respective chapters.