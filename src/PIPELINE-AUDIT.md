# PIPELINE-AUDIT.md

# Pipeline Audit

## 1. lint

**Purpose**

Runs ESLint to check the code for style and syntax issues before other jobs execute.

**Current Issue**

The repository uses ESLint 10, but only provides a `.eslintrc.json` configuration. ESLint 10 expects an `eslint.config.js` file, so the lint job fails before checking the code. The job also has no dependencies or timeout configured.

**Correct Fix**

Add a compatible `eslint.config.js` (or use a compatible ESLint version), make lint the first job in the pipeline, and configure a timeout.

---

## 2. unit-tests

**Purpose**

Runs the project's unit tests.

**Current Issue**

The job runs immediately alongside every other job instead of waiting for lint to complete. This means tests can run even when lint has already failed.

**Correct Fix**

Add `needs: lint` so unit tests only start after lint succeeds, and configure a timeout.

---

## 3. build

**Purpose**

Builds the application and creates the `dist/` directory.

**Current Issue**

The job runs without waiting for lint and does not upload the generated build artifacts. Since every GitHub Actions job runs on a separate runner, the `dist/` directory is lost after the job finishes.

**Correct Fix**

Add `needs: lint`, upload the `dist/` directory using `actions/upload-artifact@v4`, and configure a timeout.

---

## 4. integration-tests

**Purpose**

Runs integration tests using the built application.

**Current Issue**

The job does not wait for the build to complete and does not download the build artifact. Because each job uses a fresh runner, the required `dist/` directory is unavailable.

**Correct Fix**

Add `needs: build`, download the `app-build` artifact using `actions/download-artifact@v4`, and configure a timeout.

---

## 5. deploy-staging

**Purpose**

Deploys the application to the staging environment.

**Current Issue**

The job runs on every branch and does not wait for unit tests and integration tests to finish successfully.

**Correct Fix**

Run only after both testing jobs succeed and only when `github.ref == 'refs/heads/main'`. Add a timeout.

---

## 6. deploy-production

**Purpose**

Deploys the application to the production environment.

**Current Issue**

The job runs without waiting for the staging deployment and is not restricted to the main branch, allowing production deployments from feature branches.

**Correct Fix**

Add `needs: deploy-staging`, restrict execution to the `main` branch using an `if` condition, and configure a timeout.

---

## 7. notify

**Purpose**

Sends a notification indicating that the pipeline has completed.

**Current Issue**

The job does not use `if: always()`, so it may be skipped when earlier jobs fail. It also has no dependencies to ensure it runs after the rest of the pipeline.

**Correct Fix**

Use `if: always()` and make the job depend on the other pipeline jobs so it always executes after the workflow finishes.
