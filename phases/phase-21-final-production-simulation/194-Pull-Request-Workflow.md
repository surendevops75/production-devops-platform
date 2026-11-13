# PHASE 21 — FINAL PRODUCTION SIMULATION

# CHAPTER 194 — PULL REQUEST WORKFLOW

Filename: `194-Pull-Request-Workflow.md`

This chapter continues exactly where Chapter 193 ended.

Our developer has implemented the Order Tracking API, committed the changes and pushed the feature branch to GitHub. Now the developer must submit the changes for review, address feedback, satisfy repository policies and merge the approved code.

The objective is not simply to click the Merge button. It is to understand how enterprise teams control code quality, security, collaboration and traceability before application code enters the main branch.

## 1. Scenario continuity

We will continue with the same production simulation.

| Item           | Value                                     |
| -------------- | ----------------------------------------- |
| Jira issue     | `ECOM-4821`                               |
| Feature        | Order Tracking API                        |
| Microservice   | `order-service`                           |
| Repository     | `company-ecommerce/order-service`         |
| Source branch  | `feature/ECOM-4821-order-status-tracking` |
| Target branch  | `main`                                    |
| Example commit | `a1b2c3d`                                 |
| Language       | Java 21                                   |
| Framework      | Spring Boot                               |
| Build          | Maven                                     |
| Source control | GitHub                                    |

The example commit SHA is illustrative. The actual SHA must be taken from GitHub.

The continuous workflow:

```
Chapter 193
Developer Feature Workflow
        |
        v
Feature branch pushed
        |
        v
Chapter 194
Pull Request Workflow
        |
        v
PR creation
        |
        v
Automated validation
        |
        v
Code review
        |
        v
Review comments
        |
        v
Changes requested
        |
        v
Developer fixes
        |
        v
Required approvals
        |
        v
PR merge
        |
        v
Verified merge commit
        |
        v
Chapter 195
CI Pipeline Execution
```

## 2. What is a Pull Request?

A Pull Request (PR) is a request to merge changes from one Git branch into another.

For our example:

```
feature/ECOM-4821-order-status-tracking
                  |
                  | Pull Request
                  v
                 main
```

The developer is asking the team:

"Please review my implementation of the Order Tracking API and approve merging these changes into the main branch."

A PR provides a controlled place to:

- Review application code.
- Discuss implementation decisions.
- Validate automated checks.
- Identify security problems.
- Review database changes.
- Check backward compatibility.
- Resolve defects.
- Record approvals.
- Maintain an auditable history.

### 2.1 Pull Request versus Git merge

| Pull Request                           | Git merge                                     |
| -------------------------------------- | --------------------------------------------- |
| Collaboration and review workflow      | Git operation that integrates branches        |
| Usually managed through GitHub         | Can be performed using Git commands or GitHub |
| Supports review comments and approvals | Does not inherently provide review            |
| Can enforce branch policies            | Depends on repository controls                |
| Provides an audit trail                | Produces Git history                          |

A Pull Request does not automatically guarantee safe code. Its quality depends on the review process and enforced controls.

## 3. Who participates in the PR workflow?

In an enterprise environment, several people may participate.

| Role                       | Responsibility                                             |
| -------------------------- | ---------------------------------------------------------- |
| Developer                  | Creates PR and addresses feedback                          |
| Senior developer           | Reviews implementation and design                          |
| Service owner              | Checks service behavior and compatibility                  |
| DevOps / Platform Engineer | Reviews pipeline, container and deployment-related changes |
| Security Engineer          | Reviews security-sensitive changes                         |
| QA Engineer                | Reviews test coverage and acceptance criteria              |
| Code Owner                 | Reviews files owned by their team                          |
| Engineering Manager        | May approve changes under team policy                      |

Not every PR needs every reviewer. Reviewers should be selected according to the files changed and the risk of the feature.

### 3.1 Example ownership

For `ECOM-4821`:

- Order Service developer: implementation owner.
- Order Service senior developer: code reviewer.
- Platform engineer: reviewer if shared CI, Docker or Helm files are modified.
- Security reviewer: required if security-sensitive authentication or authorization code changes under company policy.
- QA engineer: validates test coverage and prepares environment testing.

The person who wrote the code should not approve their own work where separation of duties is required.

# PART 1 — PREPARE FOR THE PULL REQUEST

## 4. Verify the feature branch

Before opening a PR, the developer should verify that the correct branch is pushed.

```
git branch --show-current
```

Expected:

```
feature/ECOM-4821-order-status-tracking
```

Check repository status:

```
git status
```

Expected:

```
On branch feature/ECOM-4821-order-status-tracking
Your branch is up to date with 'origin/feature/ECOM-4821-order-status-tracking'.

nothing to commit, working tree clean
```

If there are uncommitted changes, review and commit them before opening the PR.

## 5. Verify the remote branch

```
git fetch origin
```

Check the remote branch:

```
git branch -r
```

Expected relevant entry:

```
origin/main
origin/feature/ECOM-4821-order-status-tracking
```

Check the latest commit:

```
git log -1 --oneline
```

Example:

```
a1b2c3d feat(ECOM-4821): add order tracking API
```

Inspect the commit details:

```
git show --stat HEAD
```

This helps identify which files were included in the latest commit.

## 6. Compare the feature branch with main

A developer should understand exactly what will enter the target branch.

Fetch the latest remote state:

```
git fetch origin
```

Show commits that exist on the feature branch but not on main:

```
git log --oneline origin/main..HEAD
```

Show changed files:

```
git diff --name-status origin/main...HEAD
```

Show complete changes:

```
git diff origin/main...HEAD
```

Example:

```
M src/main/java/com/company/order/controller/OrderTrackingController.java
A src/main/java/com/company/order/dto/OrderTrackingResponse.java
M src/main/java/com/company/order/service/OrderTrackingService.java
A src/test/java/com/company/order/service/OrderTrackingServiceTest.java
M README.md
```

Here:

- `M` means modified.
- `A` means added.
- `D` means deleted.

### 6.1 Why use three dots in the comparison?

This command:

```
git diff origin/main...HEAD
```

compares the feature branch against the common ancestor with the target branch.

It is useful for reviewing the changes introduced by the feature branch.

### 6.2 What should the developer look for?

- Unrelated source-code modifications.
- Accidental configuration changes.
- Debug statements.
- Hardcoded credentials.
- Test files accidentally excluded.
- Missing documentation.
- Database migrations.
- API contract modifications.
- Changes to shared libraries.
- Changes to Docker or Helm configuration.

Do not create a PR before understanding the complete diff.

## 7. Verify that the feature branch is current

Suppose another developer merged changes into `main` while our feature was under development.

Our branch might be behind.

Check:

```
git fetch origin
```

Check whether `main` has new commits:

```
git log HEAD..origin/main --oneline
```

If the team requires the latest base before review, update the branch according to its merge or rebase policy.

Example using rebase:

```
git rebase origin/main
```

If conflicts occur, resolve them carefully, run the required tests and update the remote feature branch using the team's approved safe push procedure.

```
git push --force-with-lease
```

Use this only when permitted and after considering whether other developers share the branch. Never casually overwrite someone else's remote work.

# PART 2 — CREATE THE PULL REQUEST IN GITHUB

## 8. Open the repository

Open the Order Service repository in GitHub:

```
https://github.com/company-ecommerce/order-service
```

This is an illustrative repository URL. In the real environment, use the organization's actual repository.

Navigate to:

```
Repository
   |
   v
Pull requests
   |
   v
New pull request
```

Select the branches:

Create a new Pull Request

Base branch — where changes will go

main

Compare branch — where changes currently exist

feature/ECOM-4821-order-status-tracking

Correct direction

The direction is important.

- Base is `main`.
- Compare is the feature branch.

If the developer selects the wrong direction, GitHub may show an unexpected set of changes.

## 9. Review the PR comparison before creating it

GitHub shows the differences between the branches.

The developer should inspect three areas:

Conversation: PR description and discussion.

Commits: Commits included in the PR.

Files changed: Actual code and configuration changes.

Before clicking Create Pull Request, verify:

- The correct base branch is selected.
- The correct feature branch is selected.
- The expected commits are present.
- No unrelated files are included.
- No accidental secrets are exposed.
- The changes correspond to `ECOM-4821`.

## 10. Write a professional PR title

A PR title should communicate what changed.

Recommended:

```
feat(ECOM-4821): Add Order Tracking API
```

Other examples:

```
fix(ECOM-4822): Correct payment retry handling
```

```
refactor(ECOM-4823): Improve cart calculation logic
```

```
chore(ECOM-4824): Update service dependencies
```

The Jira issue reference makes it easier to trace the implementation back to the business requirement.

Avoid unclear titles such as:

```
Changes
```

```
Latest updates
```

```
Code modifications
```

## 11. Create the PR description

A good PR description helps reviewers understand the reason for the change, what was implemented, how it was tested and what risks exist.

Example PR description:

```
## Jira Issue

ECOM-4821

## Feature

Implement Order Tracking API in order-service.

## Business Requirement

Allow authenticated customers to retrieve
tracking information for their orders.

## API Changes

GET /api/v1/orders/{orderId}/tracking

## Implementation

- Added OrderTrackingController.
- Added OrderTrackingService.
- Added OrderTrackingResponse DTO.
- Added order ownership validation.
- Added exception handling.
- Added service unit tests.

## Security

- Uses authenticated customer identity.
- Validates order ownership.
- Does not accept customer identity from
  an untrusted request parameter.
- Does not expose sensitive internal data.

## Testing

- Unit tests executed.
- Controller tests executed.
- Integration tests executed where applicable.
- Maven build completed.
- Local Docker validation completed.

## Database Changes

No database migration included.
Confirm against the actual implementation.

## Backward Compatibility

Existing Order Service APIs are unchanged.

## Deployment Notes

No direct production deployment performed.
The release follows the approved CI/GitOps process.

## Risks

External tracking dependency behavior must be
validated in the relevant integration environment.

## Reviewer Notes

Please review authentication, authorization,
API compatibility and error handling.

## Checklist

- [x] Jira issue linked
- [x] Code reviewed by author
- [x] Tests executed
- [x] Documentation updated
- [ ] Required approvals received
```

The checklist must reflect what actually happened. Do not mark tests as completed if they were not executed.

### 11.1 Why include risks?

Reviewers need to know whether a change has special considerations.

For example:

- New external dependency.
- New database migration.
- API contract modification.
- Authentication changes.
- New environment configuration.
- Feature flag dependency.

If no risks are known, say so only after the developer has assessed them.

## 12. Link the Jira issue

Our PR should reference:

```
ECOM-4821
```

Depending on the company's Jira-GitHub integration, the issue may automatically link to the PR through:

- Branch name.
- Commit message.
- PR title.
- PR description.
- Jira development panel integration.

The developer should verify that Jira actually displays the PR.

A reference in a title does not guarantee integration is configured.

### 12.1 Jira workflow

Example:

```
To Do
  |
  v
In Progress
  |
  v
In Review
  |
  v
Approved
  |
  v
Done
```

The exact Jira transitions depend on the organization's workflow.

The developer should move the issue to the correct review status according to team policy. A PR being opened does not automatically mean the feature is completed.

# PART 3 — PR REVIEWERS AND CODE OWNERS

## 13. Assign reviewers

Reviewers should be selected based on the changes.

Example:

| Reviewer                       | Reason                                            |
| ------------------------------ | ------------------------------------------------- |
| Senior Order Service Developer | Business logic and design                         |
| Order Service Owner            | Service ownership and compatibility               |
| Security Engineer              | Authentication/authorization changes, if required |
| Platform Engineer              | Shared deployment or pipeline changes, if present |
| QA Engineer                    | Test coverage and acceptance criteria             |

For our basic application change, an Order Service senior developer and service owner may be sufficient under the team's review policy.

Do not assign every engineer in the company to every PR.

## 14. What is CODEOWNERS?

CODEOWNERS is a GitHub mechanism for defining who owns specific files or directories.

For example:

```
.github/CODEOWNERS
```

Example:

```
# Order Service application
/src/main/java/ @company/order-service-team

# Application tests
/src/test/ @company/order-service-team

# Dockerfile
/Dockerfile @company/platform-team

# Helm configuration
/helm/ @company/platform-team

# GitHub Actions workflows
/.github/workflows/ @company/platform-team
```

These are illustrative GitHub usernames or teams. Replace them with real organization teams.

### 14.1 Why use CODEOWNERS?

Suppose a developer changes the Dockerfile.

The Platform team should be involved because it owns the container build standards.

Suppose a developer changes the application controller.

The Order Service team should review it.

CODEOWNERS helps route review requests to the appropriate people.

Important distinctions:

- CODEOWNERS can automatically request reviews.
- It does not itself guarantee approval.
- Required code-owner approval must be enforced through repository rules.
- The required review count and dismissal rules must be configured separately.

## 15. Review assignment versus approval

These are different activities.

| Activity          | Meaning                                                     |
| ----------------- | ----------------------------------------------------------- |
| Reviewer assigned | Person is asked to review                                   |
| Comment added     | Reviewer provides feedback                                  |
| Approved          | Reviewer approves the proposed change                       |
| Changes requested | Reviewer expects modifications                              |
| Review dismissed  | Previous review no longer counts under the applicable rules |

A PR can have several comments but still have zero valid approvals.

# PART 4 — AUTOMATED PR VALIDATION

## 16. What happens after creating the PR?

When a PR is opened or updated, GitHub Actions workflows may start automatically.

Example:

```
Developer opens PR
       |
       v
GitHub receives PR event
       |
       v
Workflow starts
       |
       v
Checkout source
       |
       v
Java setup
       |
       v
Maven build
       |
       v
Unit tests
       |
       v
Code quality checks
       |
       v
Security checks
       |
       v
Status checks reported
```

The exact checks are determined by the repository's configured workflows and organizational policy.

We will build and execute the complete pipeline in Chapter 195 — CI Pipeline Execution.

## 17. Example GitHub Actions PR trigger

Illustrative workflow:

```
name: Order Service PR Validation

on:
  pull_request:
    branches:
      - main

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Build and test
        run: ./mvnw clean verify
```

This is a minimal illustrative workflow, not the complete production pipeline.

Production workflows should also follow the organization's approved action-version pinning, permissions, runner, dependency and security policies.

### 17.1 Understand each step

Checkout source

Downloads the repository contents into the CI runner.

Setup Java

Makes the required Java version available.

Maven cache

Can reduce repeated dependency downloads.

Build and test

Executes the Maven lifecycle and the project's configured tests.

The CI pipeline can produce additional reports and status checks. Their detailed implementation belongs in Chapter 195.

## 18. What are PR status checks?

GitHub reports automated check results.

Example PR checks

Maven build

Build and test job passed

Passed

Unit tests

Configured test suite passed

Passed

SonarQube

Quality analysis running

Pending

Security scan

Example: blocking finding detected

Failed

These statuses are illustrative.

### 18.1 Required versus optional checks

Required checks

Must pass before GitHub permits a merge under the applicable branch protection or ruleset.

Optional checks

Provide information but may not prevent merging.

A check should be marked required only if the organization intends to enforce it.

A green checkmark does not automatically prove that the application is production-safe. It proves that the configured check completed successfully.

## 19. Security checks during PR validation

Our platform uses:

| Tool      | Purpose                                                                    |
| --------- | -------------------------------------------------------------------------- |
| SonarQube | Code quality and configured static analysis                                |
| Trivy     | Vulnerability scanning for supported artifacts, including container images |
| Veracode  | Application security testing                                               |

The exact execution stage depends on the organization's pipeline.

For example:

```
Pull Request
     |
     v
Application Build
     |
     v
Unit Tests
     |
     v
SonarQube Analysis
     |
     v
Security Analysis
     |
     v
Required Status Checks
```

Trivy image scanning may occur when an image is built, rather than at the earliest PR stage. The detailed security and image-build flow will be covered in Chapters 196 and 197.

## 20. What if a security scan fails?

Example:

```
Security finding:
Potential hardcoded credential
```

The developer should:

1. Read the finding.
2. Identify the affected file.
3. Determine whether an actual secret was committed.
4. If a real credential was exposed, revoke or rotate it immediately.
5. Remove the secret from the source.
6. Review whether repository history must be cleaned.
7. Add a regression check.
8. Push the corrected change.
9. Wait for the required security checks to complete again.

Never assume deleting a secret from the latest file is sufficient. It may still exist in previous commits.

# PART 5 — REVIEW THE APPLICATION CODE

## 21. How should reviewers inspect the implementation?

The reviewer should understand the business requirement before reviewing the source.

For `ECOM-4821`, the reviewer should ask:

- Does the API meet the acceptance criteria?
- Does the API follow existing conventions?
- Is the correct service responsible?
- Is customer ownership enforced?
- Is sensitive data protected?
- Are errors handled consistently?
- Are tests sufficient?
- Is the implementation compatible with existing consumers?
- Does the change introduce operational risks?

## 22. Review the controller

Example:

```
@GetMapping("/{orderId}/tracking")
public OrderTrackingResponse getTracking(
        @PathVariable String orderId,
        @AuthenticationPrincipal Jwt jwt
) {

    String customerId = jwt.getSubject();

    return orderTrackingService.getTracking(
            orderId,
            customerId
    );
}
```

Reviewer questions:

1. Is the endpoint naming consistent?
2. Is authentication enforced by the application?
3. Is the principal obtained from a verified security context?
4. Is the customer identifier mapped correctly?
5. Is authorization enforced?
6. Are invalid inputs handled?
7. Is the response contract documented?

A reviewer should not approve this code merely because it compiles.

The reviewer must confirm that the real authentication configuration establishes the principal and that the `sub` claim has the expected meaning.

## 23. Review the service layer

Example:

```
Order order = orderRepository
        .findByOrderIdAndCustomerId(orderId, customerId)
        .orElseThrow(() ->
                new OrderNotFoundException(orderId)
        );
```

Reviewer questions:

- Does this query enforce ownership?
- Is the database access efficient?
- Does the repository match the existing entity?
- Is the not-found behavior consistent?
- Does the code leak customer information?
- Is a separate integration service required?

A security-sensitive reviewer should pay particular attention to whether authorization is performed using trusted identity information.

## 24. Review the DTO

Example:

```
public record OrderTrackingResponse(
        String orderId,
        String status,
        String trackingNumber,
        String carrier,
        LocalDate estimatedDelivery,
        Instant lastUpdated
) {
}
```

Reviewer questions:

- Are all fields required?
- Are any sensitive fields exposed?
- Are date formats consistent?
- Are nullable values handled?
- Does the API contract match the documentation?
- Are fields backward-compatible?

## 25. Review the tests

The reviewer should inspect test cases, not just test counts.

For example:

| Test case                                  | Expected result                         |
| ------------------------------------------ | --------------------------------------- |
| Customer retrieves own order               | Allowed                                 |
| Customer requests another customer's order | Denied or concealed according to policy |
| Order does not exist                       | Correct not-found behavior              |
| Tracking unavailable                       | Agreed pending behavior                 |
| Invalid order ID                           | Validation error                        |
| Missing authentication                     | Unauthorized                            |
| Existing order APIs                        | Regression tests pass                   |

Tests should verify actual application behavior.

A test that only verifies the controller returns HTTP 200 without testing authorization is insufficient for this feature.

## 26. Review database changes

If the PR contains a migration, inspect:

- Whether the correct service owns the table.
- Whether the migration is backward-compatible.
- Whether existing data is preserved.
- Whether indexes are needed.
- Whether the migration locks a large table.
- Whether application versions can coexist.
- Whether a rollback or forward-recovery strategy exists.

If the PR contains no database migration, verify that this matches the actual implementation.

Do not approve a schema change simply because the SQL executes successfully on a local database.

## 27. Review configuration and Docker changes

If the PR modifies a Dockerfile, inspect:

- Base image.
- Runtime image.
- Build and runtime separation.
- Container user.
- Exposed port.
- Secret handling.
- Unnecessary packages.
- Image size.
- Startup command.

If Helm files are modified, inspect:

- Image reference.
- Resource requests and limits.
- Probes.
- Service account.
- Configuration.
- Security context.
- Environment-specific values.

The same review principles apply to shared GitHub Actions workflows and infrastructure changes.

## 28. Review logging and monitoring

For this feature, reviewers should consider:

- Are important errors logged?
- Are logs structured?
- Is correlation information preserved?
- Are secrets excluded?
- Are customer identifiers handled according to data policy?
- Could metric labels create high cardinality?
- Are new dependencies observable?

The goal is to make the feature operable, not merely functional.

# PART 6 — REVIEW COMMENTS AND CHANGES REQUESTED

## 29. How does a reviewer leave a comment?

GitHub supports:

- General PR comments.
- Inline comments on individual lines.
- Review summaries.
- Approval.
- Request changes.

Example inline review comment:

```
File:
OrderTrackingController.java

Comment:

Please verify that the customer identifier
comes from the application's validated security
principal and not from an untrusted request value.

Also add a test that attempts to access
another customer's order.
```

A good review comment should explain the problem and expected improvement.

Avoid vague comments such as:

```
Fix this.
```

## 30. Example: Reviewer requests changes

Suppose the senior developer identifies a missing authorization test.

Review status:

Pull Request review

Changes requested

Reviewer: Order Service Senior Developer

Feedback:

- Add a test for cross-customer order access.
- Confirm that customer identity comes from the authenticated principal.
- Verify the error response matches the existing API conventions.

Example review feedback for ECOM-4821

The developer must address the comments before the PR can satisfy the review policy.

## 31. Developer fixes the code

The developer returns to the feature branch.

```
git checkout feature/ECOM-4821-order-status-tracking
```

Check current state:

```
git status
```

Make the required code changes.

Add the authorization test.

Run:

```
./mvnw clean verify
```

Review the changes:

```
git diff
```

Stage and commit:

```
git add src/test/
git add src/main/
```

Commit:

```
git commit -m "test(ECOM-4821): verify order ownership"
```

Push:

```
git push origin feature/ECOM-4821-order-status-tracking
```

The existing PR is updated automatically because it tracks this source branch.

Important: The developer normally does not need to create a second PR for the same feature.

## 32. What happens after the new commit?

```
Developer pushes changes
        |
        v
Existing PR updated
        |
        v
New commit appears
        |
        v
CI checks run again
        |
        v
Reviewers inspect updated code
        |
        v
Review comments addressed
        |
        v
Approval process continues
```

GitHub may invalidate approvals under configured stale-review dismissal rules when the branch changes. The developer should check the PR's current review state and repository policy.

## 33. Reply to review comments

Example:

```
Thanks for the review.

I have added a test covering access to another
customer's order and verified that the customer
identity is obtained from the authenticated
security principal.

The updated tests have passed.

Please review the latest commit.
```

Do not simply reply "Done" without making the requested change.

## 34. Resolve review conversations

A conversation should be marked resolved only after the requested change has been addressed or the reviewer has agreed that the issue is no longer applicable.

The developer should not resolve a reviewer conversation just to make the PR look clean.

The reviewer should be able to verify the actual fix.

# PART 7 — MERGE CONFLICTS

## 35. What is a merge conflict?

A merge conflict happens when Git cannot automatically combine changes from different branches.

Example:

Developer A modifies:

```
public String getOrderStatus() {
    return "SHIPPED";
}
```

Developer B modifies the same section:

```
public String getOrderStatus() {
    return "DISPATCHED";
}
```

Git may not know which change should remain.

## 36. Example conflict

When a conflict occurs, Git may show:

```
<<<<<<< HEAD
public String getOrderStatus() {
    return "SHIPPED";
}
=======
public String getOrderStatus() {
    return "DISPATCHED";
}
>>>>>>> origin/main
```

The conflict markers mean:

- `<<<<<<<` starts one version.
- `=======` separates the versions.
- `>>>>>>>` ends the other version.

The developer must understand the business meaning before resolving the conflict.

Do not blindly accept one side.

## 37. Resolve conflict using rebase

First fetch the latest main:

```
git fetch origin
```

Rebase:

```
git rebase origin/main
```

Git stops at conflicting files.

Check:

```
git status
```

Open the conflicting files and resolve them.

After resolution:

```
git add src/main/java/com/company/order/service/OrderTrackingService.java
```

Continue:

```
git rebase --continue
```

Repeat if other conflicts appear.

If the rebase becomes unsafe or the developer needs to restart:

```
git rebase --abort
```

After a successful rebase, run tests:

```
./mvnw clean verify
```

Then update the remote feature branch using the approved safe push process:

```
git push --force-with-lease origin feature/ECOM-4821-order-status-tracking
```

This is a history rewrite, so coordinate with other developers who may be using the branch.

## 38. Resolve conflict using merge instead

If the team prefers merge commits:

```
git fetch origin
```

Merge main into the feature branch:

```
git merge origin/main
```

Resolve conflicts, then:

```
git add .
git commit
```

Run tests:

```
./mvnw clean verify
```

Push:

```
git push origin feature/ECOM-4821-order-status-tracking
```

### 38.1 Merge versus rebase

| Rebase                           | Merge                           |
| -------------------------------- | ------------------------------- |
| Replays commits on a new base    | Combines branch histories       |
| Creates a linear-looking history | May create a merge commit       |
| Rewrites commit IDs              | Preserves existing commits      |
| Requires care on shared branches | Often safer for shared branches |

The team should use a consistent strategy.

# PART 8 — GITHUB BRANCH PROTECTION AND MERGE POLICIES

## 39. Why protect the main branch?

Imagine a developer accidentally pushes untested code directly to `main`.

If `main` is the source for the next release pipeline, that change could enter the release process without review.

Branch protection reduces this risk.

A protected branch can require:

- Pull Requests.
- Minimum approvals.
- Code-owner approval.
- Passing status checks.
- Resolved review conversations.
- Up-to-date branches.
- Restrictions on force pushes.
- Restrictions on branch deletion.

The exact settings depend on the organization's GitHub plan and repository policy.

## 40. Configure branch protection

Repository administrators navigate to:

```
GitHub Repository
      |
      v
Settings
      |
      v
Branches
      |
      v
Branch protection rules
```

Alternatively, organizations using GitHub rulesets can configure equivalent controls through the Rules section.

Example rule:

Example `main` branch policy

Require a pull request before merging

Enabled

Required approvals\
2

Require code-owner review

Enabled

Require status checks

Enabled

Require resolved conversations

Enabled

Require up-to-date branch

Enabled

Allow force pushes

Disabled

Allow branch deletion

Disabled

Example policy only. Actual approval counts and exceptions must be defined by the organization.

### 40.1 Require a Pull Request

This prevents normal direct pushes to the protected branch.

Changes must enter through the approved PR process.

### 40.2 Require approvals

For example, two approvals may be required.

The number is a policy decision, not a universal requirement.

A production-critical application may have stricter rules than a small internal utility.

### 40.3 Require code-owner approval

If the PR modifies files owned by the Platform team, GitHub can require approval from the relevant code owners.

### 40.4 Require status checks

Administrators select the exact check names that must pass.

For example:

```
Maven Build
Unit Tests
SonarQube Quality Gate
Security Validation
```

These names are illustrative. Select the actual checks reported by the repository workflows.

Do not select a check name that is never produced by the configured workflow.

### 40.5 Require conversation resolution

This helps ensure that review feedback is addressed before merging.

It does not replace the need for a reviewer to verify the correction.

### 40.6 Restrict force pushes

A protected `main` branch should normally prevent force pushes by ordinary contributors.

Force pushing can rewrite shared history and remove commits from the visible branch history.

### 40.7 Restrict branch deletion

The main integration branch should not be casually deleted.

Use organization-controlled administration for exceptional maintenance.

## 41. What happens if someone tries to bypass the policy?

Suppose a developer runs:

```
git push origin main
```

GitHub may reject the operation because the user is not authorized to push directly to the protected branch.

Example:

```
remote: error: GH006: Protected branch update failed
remote: Changes must be made through a pull request.
```

The precise error depends on the configured rules and GitHub enforcement mechanism.

The correct approach is to create a feature branch and submit a PR.

## 42. Separation of duties

Production-grade repositories should avoid allowing one person to perform every sensitive action without oversight.

Example:

```
Developer
   |
   | Creates PR
   v
Senior Developer
   |
   | Reviews code
   v
Code Owner
   |
   | Approves owned changes
   v
CI Validation
   |
   | Passes required checks
   v
Authorized Merger
   |
   | Merges PR
   v
main
```

Some organizations allow the PR author to merge after approvals. Others require a separate authorized merger.

The organization's policy should define this clearly.

# PART 9 — MERGE STRATEGIES

## 43. Understand the three common merge strategies

GitHub commonly supports three merge methods, subject to repository configuration.

### 43.1 Create a merge commit

This preserves the feature branch's individual commits and creates a merge commit.

Illustration:

```
A---B---C-------------M  main
     \               /
      D---E---F------
```

Advantages:

- Preserves branch history.
- Shows the integration point.
- Maintains individual feature commits.

Disadvantages:

- Can make the history more complex.
- May create unnecessary merge commits for very small changes.

### 43.2 Squash and merge

Squash combines the feature branch commits into one commit on the target branch.

Illustration:

```
A---B---C---S  main
     \
      D---E---F
```

The feature work becomes one consolidated commit, `S`, on `main`.

Advantages:

- Cleaner main branch history.
- Easier to identify a feature as one change.
- Avoids many small intermediate commits.

Disadvantages:

- Individual commit history is not retained on the target branch as separate commits.
- Commit messages must be meaningful.

For our example, a squash commit could be:

```
feat(ECOM-4821): add order tracking API
```

### 43.3 Rebase and merge

Rebase and merge applies feature commits on top of the target branch without a merge commit.

Illustration:

```
A---B---C---D'---E'---F'  main
```

The rebased commits have new identities.

Advantages:

- Linear history.
- Individual feature commits remain visible.

Disadvantages:

- Rewrites commit identities.
- Requires careful team understanding of history.

## 44. Which strategy should our project use?

For this simulation, we will use Squash and merge as an illustrative team convention.

Reason:

- One Jira story represents one feature.
- The target branch remains easy to read.
- The resulting commit can reference `ECOM-4821`.
- The merged feature can be traced to the PR.

This is a project convention for our example, not a universal recommendation.

## 45. Check the PR before merging

The authorized merger should verify:

- Correct base branch.
- Correct feature branch.
- Jira issue linked.
- Required approvals received.
- Code-owner approvals received where required.
- Required CI checks passed.
- No unresolved blocking conversations.
- Security findings addressed.
- Database changes reviewed.
- Merge conflicts resolved.
- Latest source changes reviewed.
- Merge strategy follows team policy.

Do not merge simply because GitHub displays a green merge button.

## 46. Perform the merge

After all required conditions are satisfied, the authorized person selects:

```
Squash and merge
```

GitHub displays a confirmation dialog.

Example merge commit message:

```
feat(ECOM-4821): add order tracking API
```

The user confirms the merge.

GitHub then integrates the change into `main`.

PR #4821

Merged

feat(ECOM-4821): Add Order Tracking API

Base branch

main

Source branch

feature/ECOM-4821-order-status-tracking

Merge strategy

Squash and merge

Jira issue

ECOM-4821

Illustrative PR details, not a real GitHub operation.

# PART 10 — VERIFY THE MERGED CHANGE

## 47. Update local main

After the PR is merged, the developer should synchronize the local repository.

```
git checkout main
```

Fetch remote changes:

```
git fetch origin
```

Pull the latest main:

```
git pull --ff-only origin main
```

Check the latest commit:

```
git log -1 --oneline
```

Example:

```
d4e5f6a feat(ECOM-4821): add order tracking API
```

The example merge SHA is different from the feature commit SHA because squash merging creates a new commit on `main`.

### 47.1 Verify the commit

```
git show --stat d4e5f6a
```

Check the commit metadata:

```
git show --no-patch --format=fuller d4e5f6a
```

Verify that the merged files match the PR.

```
git show --name-status d4e5f6a
```

Do not rely on an illustrative SHA. Always capture the real merge commit from the repository.

## 48. Verify the commit in GitHub

Navigate to:

```
Repository
    |
    v
Code
    |
    v
Commits
```

Confirm:

- Commit message.
- Commit SHA.
- Author and committer information.
- PR reference, where available.
- Expected files changed.

This establishes the source-code version that the next CI stage should build.

## 49. Delete the feature branch

After successful merge, GitHub may offer to delete the feature branch.

For a completed feature, deleting the branch is usually appropriate if the team does not need to keep it.

The developer can also delete it locally:

```
git branch -d feature/ECOM-4821-order-status-tracking
```

Delete the remote branch if it was not already removed by GitHub:

```
git push origin --delete feature/ECOM-4821-order-status-tracking
```

Do not delete a branch that still contains unmerged work.

## 50. Update Jira

The developer or Jira integration should update the issue according to the workflow.

Example:

```
ECOM-4821

Status: In Review -> Done
```

However, the issue should move to Done only if the team's Definition of Done permits that transition at the merge stage.

If QA acceptance is still pending, the issue may remain in a review or testing state.

The PR merge does not mean that the feature is already deployed to production.

# PART 11 — COMMON PR FAILURES AND TROUBLESHOOTING

## 51. Failure: Wrong base branch selected

Symptom: GitHub displays unexpected changes or shows a large number of commits.

Possible causes:

- Wrong base branch.
- Feature branch created from an incorrect branch.
- Incorrect compare branch.
- Branch history divergence.

Troubleshooting:

```
git branch -vv
git fetch origin
git log --oneline --graph --decorate --all
git diff --name-status origin/main...HEAD
```

Fix:

1. Confirm the intended target branch.
2. Confirm the branch's common history.
3. Correct the PR base if appropriate.
4. Re-review the entire comparison.

Do not merge a PR whose source and target branches have not been verified.

## 52. Failure: Required status check is missing

Symptom: PR cannot merge because a required check has not been reported.

Possible causes:

- Workflow did not trigger.
- Workflow file has invalid YAML.
- Required check name does not match.
- Workflow was skipped.
- Check was not created for this event.
- CI runner failed to start.

Troubleshooting:

1. Open the PR Checks tab.
2. Inspect workflow execution.
3. Review the Actions tab.
4. Confirm the workflow trigger.
5. Verify required check names in branch protection.
6. Correct the workflow or branch policy.

Do not remove a required security check merely to merge the PR.

## 53. Failure: Unit tests fail

Example:

```
Tests run: 25
Failures: 2
Errors: 0
```

Troubleshooting:

```
./mvnw test
```

Inspect Maven reports:

```
find target/surefire-reports -type f
```

Read a report:

```
cat target/surefire-reports/*.txt
```

Possible causes:

- Incorrect expected response.
- Test data problem.
- Missing mock setup.
- Changed application behavior.
- Dependency configuration.
- Environment-specific test assumptions.

Fix the root cause, then rerun the relevant test suite and full validation.

## 54. Failure: SonarQube quality gate fails

Possible reasons:

- New code has security issues.
- Code smells exceed policy.
- Coverage is insufficient.
- Duplicated code is above the threshold.
- Analysis configuration is incorrect.

Correct approach:

1. Open the quality report.
2. Identify findings on new code.
3. Review the actual implementation.
4. Fix the relevant issue.
5. Rerun analysis.
6. Confirm the updated quality gate.

Do not treat all SonarQube findings as equally severe. Follow the configured quality policy and prioritize security and correctness.

## 55. Failure: Reviewer is not approving

Possible reasons:

- Missing test coverage.
- Unclear implementation.
- Security concerns.
- Incomplete documentation.
- API compatibility problem.
- Unresolved review comments.

Correct approach:

1. Read the review feedback.
2. Ask for clarification if required.
3. Implement the requested correction.
4. Add tests.
5. Push an updated commit.
6. Request another review.

Do not pressure reviewers to approve code that still has unresolved defects.

## 56. Failure: PR has merge conflicts

Check:

```
git fetch origin
git status
```

Then update the feature branch according to the team strategy.

Rebase:

```
git rebase origin/main
```

Or merge:

```
git merge origin/main
```

Resolve, validate and push the corrected branch.

## 57. Failure: Approval disappears after new commits

Some repositories dismiss stale approvals when new commits are added.

Possible sequence:

```
PR approved
    |
    v
Developer pushes another commit
    |
    v
Previous approval dismissed
    |
    v
New review required
```

Correct approach:

1. Inspect the PR review state.
2. Verify required approvals.
3. Request another review if necessary.
4. Wait for the required approval before merging.

This is expected behavior under certain repository policies, not necessarily a GitHub error.

## 58. Failure: Accidental secret committed

If a developer commits a secret:

1. Treat it as exposed.
2. Revoke or rotate the credential.
3. Notify the security team.
4. Investigate exposure.
5. Remove the secret from the current code.
6. Follow repository-history cleanup procedures.
7. Run security scanning again.
8. Verify the replacement secret is stored securely.

Deleting the secret from the latest commit does not invalidate an exposed credential.

## 59. Failure: PR merged but main build fails

This is an important production scenario.

A PR may pass its own checks but the resulting main-branch build may fail because:

- Main branch changed.
- A check ran against a different merge state.
- A dependency or external service changed.
- A workflow is configured differently for push events.
- A test relies on non-isolated shared state.

Correct approach:

1. Stop the next release progression if the failure affects release safety.
2. Inspect the main-branch CI run.
3. Identify the failing commit and test.
4. Determine whether the problem is caused by the merge.
5. Fix forward or revert according to team policy.
6. Re-run the full validation.

Chapter 195 covers the CI pipeline execution and its recovery procedures.

# PART 12 — ADVANCED ENTERPRISE PR CONTROLS

## 60. Prevent unsafe GitHub Actions execution

Pull Request workflows execute code from branches. This means the CI workflow itself is part of the security boundary.

For example, a developer could modify:

```
.github/workflows/build.yml
```

A malicious or accidental change could attempt to expose credentials or perform unauthorized operations.

Important controls:

- Give workflows only the permissions they need.
- Do not expose production secrets to untrusted PR code.
- Protect sensitive environments.
- Restrict who can approve and deploy.
- Review workflow modifications as carefully as application code.
- Use trusted, approved actions and runners.
- Pin third-party actions according to organizational policy.

Example restricted permissions:

```
permissions:
  contents: read
```

If a workflow only needs to read the repository, it should not receive write access.

Avoid:

```
permissions: write-all
```

unless there is a documented and justified requirement.

### 60.1 Pull Requests from forks

A fork is a separate copy of a repository.

External contributors may submit PRs from their own forks.

This creates additional security considerations:

- The source branch is not controlled by the organization.
- Workflow files may be modified.
- Code executed in the workflow may be untrusted.
- Repository secrets must be protected.

Be particularly careful with `pull_request_target`. It runs with access associated with the base repository's context. Never combine privileged execution with checking out and executing untrusted PR code without a carefully designed security boundary.

For our internal enterprise simulation, the developer is working within the company's repository. Nevertheless, the platform team should understand fork-based PR security because the same repository may later accept external contributions.

## 61. GitHub Apps and automation permissions

Our later CI/CD workflows may need to:

- Read repository code.
- Publish build status.
- Create or update a GitOps PR.
- Publish release metadata.
- Update an authorized repository.

Use narrowly scoped GitHub Apps or other approved credentials.

For example:

| Permission           | Example requirement             |
| -------------------- | ------------------------------- |
| Contents: read       | Checkout source                 |
| Pull requests: write | Create or update PRs            |
| Checks: write        | Publish check results           |
| Contents: write      | Update an authorized repository |

Do not give every workflow all these permissions.

A developer's personal access token should not be used as a permanent organization-wide automation credential.

The GitOps promotion workflow will be discussed in Chapter 199.

## 62. Merge queue and concurrent PRs

Suppose two PRs are ready to merge:

```
PR #4821 — Order Tracking API
PR #4822 — Payment Retry Improvement
```

Both passed checks against a similar version of `main`.

However, merging PR #4821 can change the code against which PR #4822 was validated.

A merge queue can validate PRs against the latest target-branch state before merging them.

Illustration:

```
PR #4821 ----\
              \
               v
           Merge Queue
               |
               v
        Validate combined state
               |
               v
             main
               ^
               |
PR #4822 ----/
```

A merge queue is useful when repositories have frequent concurrent changes.

It is not mandatory for every repository. Use it where the team's GitHub configuration and development volume justify it.

## 63. Reverting a merged Pull Request

Suppose `ECOM-4821` is merged, but a serious defect is discovered before the feature is deployed.

If the merge used squash and merge, the resulting feature can usually be reverted using the merge commit.

Find the commit:

```
git log --oneline
```

Example:

```
d4e5f6a feat(ECOM-4821): add order tracking API
```

Create a separate revert branch:

```
git checkout main
git pull --ff-only origin main
git checkout -b revert/ECOM-4821-order-tracking
```

Revert the commit:

```
git revert d4e5f6a
```

Push the revert branch:

```
git push -u origin revert/ECOM-4821-order-tracking
```

Then create a PR for the revert.

Important: Do not reset the shared `main` branch to remove a commit. Reverting creates a new commit that records the reversal and preserves shared history.

Also remember that reverting application code does not automatically reverse a database migration or undo external side effects. Database and data recovery require separate planning.

If the issue is already deployed, the production rollback and recovery process is covered in Chapters 209 and 210.

# PART 13 — PR AUDIT AND TRACEABILITY

## 64. Why is PR traceability important?

In a production environment, we must be able to answer:

- Who requested the feature?
- Who implemented it?
- Who reviewed it?
- Which automated checks ran?
- Which commit was approved?
- Who merged it?
- Which source revision entered the main branch?
- Which release eventually deployed it?

This is essential for troubleshooting, audit, security investigations and incident analysis.

## 65. Trace the ECOM-4821 change

Our traceability chain begins here:

```
Jira Issue
ECOM-4821
     |
     v
Git Feature Branch
feature/ECOM-4821-order-status-tracking
     |
     v
Feature Commit
a1b2c3d
     |
     v
GitHub Pull Request
PR #4821
     |
     v
Code Reviews
Required approvals
     |
     v
Automated Checks
Required checks passed
     |
     v
Squash Merge
     |
     v
Main Commit
d4e5f6a
```

The IDs and SHAs are illustrative. The actual values must be recorded from the corresponding systems.

## 66. What evidence should be retained?

| Evidence             | Source                  |
| -------------------- | ----------------------- |
| Business requirement | Jira                    |
| Feature branch       | GitHub                  |
| Original commits     | GitHub                  |
| PR discussion        | GitHub                  |
| Review approvals     | GitHub                  |
| CI status            | GitHub Actions          |
| Security results     | Approved scanning tools |
| Merge commit         | GitHub                  |
| Release association  | CI/CD release process   |

The PR should remain available according to the company's source-control retention and audit policies.

## 67. What should happen to the Jira ticket?

After the PR is merged, the developer updates the Jira ticket according to the team's workflow.

A useful comment:

```
Implementation completed and merged into main.

Pull Request: #4821
Feature branch: feature/ECOM-4821-order-status-tracking

The change is now available in the main branch.
It will continue through the approved CI/CD process.

Production deployment has not yet been completed.
```

This distinction prevents business stakeholders from assuming that merged code is already live.

# PART 14 — HANDOFF TO CI/CD

## 68. What changes after the PR is merged?

The application source code is now part of `main`.

The next stage is CI pipeline execution.

Example:

```
GitHub Pull Request
       |
       v
Review and Approval
       |
       v
PR Merge
       |
       v
main branch updated
       |
       v
Push event
       |
       v
CI Pipeline Trigger
       |
       v
Build and Test
       |
       v
Security Scanning
       |
       v
Docker Image Build
```

The precise pipeline trigger and build strategy are defined by the repository workflow.

### 68.1 What does CI receive?

The pipeline must build the actual merged source revision.

For example:

```
Repository: company-ecommerce/order-service

Branch: main

Commit: d4e5f6a
```

The CI system should record the exact commit SHA rather than relying only on a branch name.

### 68.2 Why is the merge commit important?

The feature branch may have passed tests before merging.

However, the main branch now contains the merged result.

The CI pipeline must validate that exact main-branch revision.

This prevents confusion about whether the tests were run against the actual source revision that entered the release process.

### 68.3 What happens to the image tag later?

In Chapter 197, we will build the Docker image.

Our image tag will be derived from the actual source commit SHA.

Illustrative format:

```
order-service:d4e5f6a
```

Later, the registry will return an immutable image digest.

That digest will be used to identify the exact image promoted through environments.

We must not rebuild a different image for QA, staging or production.

# PART 15 — PR COMPLETION CHECKLIST

## 69. Pull Request readiness checklist

PR completion

# 0%

0 of 31 items verified

Correct repository selected

Correct base branch selected

Correct feature branch selected

Latest relevant main changes considered

All expected commits included

Complete file diff reviewed

PR title follows naming convention

Jira issue ECOM-4821 linked

Business requirement documented

Implementation summary provided

API changes documented

Authentication and authorization reviewed

Database impact reviewed

Backward compatibility reviewed

Tests and validation evidence included

Security impact reviewed

Dependencies reviewed

Docker and Helm changes reviewed if applicable

Risks and deployment notes documented

Appropriate reviewers assigned

CODEOWNERS requirements satisfied

Required status checks passed

Review comments addressed

Required approvals received

All blocking conversations resolved

Merge conflicts resolved

Latest changes validated

Authorized merge strategy selected

Merge commit SHA recorded

Jira status updated correctly

CI handoff information verified

# PART 16 — INTERVIEW PREPARATION

## 70. Scenario-based Pull Request interview questions

### Q1. A developer has pushed code to a feature branch. What happens next?

The developer opens a Pull Request against the correct target branch. The PR is linked to Jira, reviewers are assigned, automated checks execute, and the code goes through review and approval before merging.

### Q2. What is the difference between a PR and a merge?

A PR is a collaboration and review process. A merge is the Git operation that integrates one branch into another.

### Q3. How do you enforce code review before merging?

Configure branch protection or repository rules requiring a PR, approvals, code-owner reviews where applicable, required status checks and resolved review conversations.

### Q4. What happens if CI fails on a PR?

Inspect the failing job and logs, identify the root cause, correct the code or configuration, push an update and allow the required checks to run again.

### Q5. What happens if a reviewer requests changes?

The developer reviews the feedback, updates the feature branch, adds or modifies tests, pushes the new commit and requests another review.

### Q6. What is CODEOWNERS?

It is a GitHub mechanism for defining file and directory ownership and automatically requesting reviews from designated owners. Required code-owner approval must also be enforced through repository rules.

### Q7. What is the difference between squash and merge and rebase and merge?

Squash combines feature commits into one commit on the target branch. Rebase and merge replays the feature commits on top of the target branch without a merge commit.

### Q8. Why should force pushes be disabled on main?

Force pushes rewrite branch history and can remove shared commits. Protecting main preserves reliable history and prevents unauthorized history changes.

### Q9. What is a merge conflict?

It occurs when Git cannot automatically reconcile competing changes. Developers must inspect the conflicting code, resolve it correctly and run validation.

### Q10. Why should security checks be required before merging?

They help prevent vulnerable or non-compliant code from entering the main branch. The required checks should reflect the organization's security policy.

### Q11. A PR is approved, but the developer pushes a new commit. What can happen?

Depending on the repository's stale-review rules, previous approvals may be dismissed and new approval may be required.

### Q12. What is the difference between merging and deploying?

Merging integrates source code into a branch. Deploying releases an application artifact into an environment. These are separate processes.

### Q13. What would you do if a PR was merged accidentally?

Assess the impact, stop further release progression if necessary, create a controlled revert PR and verify the resulting main-branch state. If already deployed, follow the production incident and rollback procedures.

### Q14. How do you trace a production defect to its PR?

Start with the deployed application version, image digest and source commit. Trace the commit to the GitHub PR, its review history, CI execution and Jira requirement.

### Q15. Why should a CI pipeline build the merged main commit?

Because the merged revision is the actual source entering the next delivery stage. It must be validated independently of assumptions about the earlier feature-branch checks.

# PART 17 — CHAPTER 194 FINAL RESULT

## 71. Final workflow result

We have continued the same `ECOM-4821` scenario.

Chapter 194 outcome

Simulation complete

Jira issue

ECOM-4821

Pull Request

\#4821 (example)

Source branch

Feature branch

Target branch

main

Review

Approved in simulation

Merge strategy

Squash and merge

Example merge SHA

d4e5f6a

No real GitHub repository was modified. The workflow and identifiers above are illustrative.

### 72. What has been completed?

- Developer feature implementation from Chapter 193.
- PR creation and description.
- Jira linkage.
- Reviewer assignment.
- CODEOWNERS understanding.
- PR automation and required checks.
- Application and security review.
- Review comments and fixes.
- Merge conflict handling.
- Branch protection.
- Merge strategies.
- Controlled merge and verification.
- Jira and CI handoff.
- PR failure troubleshooting.
- Audit and traceability.

### 73. What has not happened yet?

We have not:

- Executed the complete CI pipeline.
- Run the full SonarQube quality-gate implementation.
- Executed the complete Trivy and Veracode security stages.
- Built the production Docker image.
- Published the image to ECR.
- Updated GitOps repositories.
- Deployed the service to any environment.

Those are subsequent chapters.

# NEXT — CHAPTER 195

# CI PIPELINE EXECUTION

Filename: `195-CI-Pipeline-Execution.md`

We will continue with the same merged Order Service commit:

```
Jira: ECOM-4821

Repository:
company-ecommerce/order-service

Branch:
main

Merge commit:
d4e5f6a
```

Chapter 195 will cover:

1. What happens immediately after the PR merge.
2. GitHub push events and workflow triggers.
3. Jenkins versus GitHub Actions versus GitLab CI/CD responsibilities.
4. CI runner architecture and permissions.
5. Repository checkout and exact source revision.
6. Maven build lifecycle.
7. Dependency caching.
8. Unit and integration testing.
9. Build artifacts and test reports.
10. SonarQube integration and quality gates.
11. CI secrets and credential management.
12. Pipeline concurrency and cancellation.
13. Build failures and retry strategies.
14. Pipeline notifications and audit records.
15. CI performance optimization.
16. Secure pipeline design.
17. Troubleshooting production-style pipeline failures.
18. Complete validation and interview scenarios.

Important continuity rule: Chapter 195 must validate the merged main-branch revision, not go back to the original feature branch. We will capture the real CI run ID and source SHA as the next traceability records.