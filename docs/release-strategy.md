# Release and Deployment Strategy

This document describes how changes flow from development into production. It focuses on branching, tagging, image promotion, and database changes, consistent with the design in `README.md`.

---

## 1. Branch Strategy

The branch model is intentionally simple and suited to a small-to-medium internal platform team.

| Branch / Tag   | Purpose |
|----------------|---------|
| `feature/*`    | Short-lived branches for individual changes or features |
| `main`         | Integration branch; always in a deployable state and mapped to dev |
| `release/*`    | Release tags that represent candidates for staging and production |

Guidelines:

- All work is done in `feature/*` branches and merged via pull requests into `main`.
- `main` is protected (required reviews, status checks).
- Release tags (e.g. `release/1.4.0`) are created from `main` once dev validation looks good.

---

## 2. Pull Request Validation

For every pull request targeting `main`:

1. **Linting**: Static analysis of code, YAML, and Dockerfiles.
2. **Unit tests**: Fast feedback on functional correctness.
3. **Optional integration tests (mocked)**: Where feasible, tests relying on lightweight or mocked services.

A pull request cannot be merged unless all required checks pass. This keeps `main` in a deployable state and reduces the risk of broken builds being deployed to dev.

---

## 3. Image Tagging Strategy

Images are tagged in a way that supports traceability and safe promotion.

| Context        | Tag Format          | Example           |
|----------------|---------------------|-------------------|
| Dev build      | `sha-<short-sha>`   | `sha-a1b2c3d`     |
| Release build  | `release-<version>` | `release-1.4.0`   |
| Prod deployment | Same as staging     | `release-1.4.0`   |

Key points:

- Each merge to `main` triggers a build; the image is tagged with the corresponding commit SHA.
- When a release tag (e.g. `release/1.4.0`) is created, the pipeline re-tags the vetted dev image as `release-<version>`.
- Production deployments use the **same** `release-<version>` image that was validated in staging.

---

## 4. Deployment Flow

The intended deployment flow aligns environments with branch and tag events.

### 4.1 `main` → dev

Trigger: Merge to `main`.

1. Pipeline builds a new image with tag `sha-<short-sha>`.
2. Image is pushed to the container registry.
3. Helm deployment runs against the `platform-dev` namespace.
4. Smoke and integration tests run against `dev.sornsub.online`.

### 4.2 Release tag or staging branch → staging

Trigger: Push of a `release/<version>` tag (from `main`).

1. Pipeline identifies the corresponding `sha-<short-sha>` image.
2. Image is re-tagged as `release-<version>`.
3. Helm deployment runs against the `platform-staging` namespace with the `release-<version>` tag.
4. Acceptance and regression tests run against `staging.sornsub.online`.

If your workflow prefers a long-lived `staging` branch, the same concept can apply: merges to `staging` would be treated similarly to a release event.

### 4.3 Manual approval → prod

Trigger: Manual approval step in the pipeline (after staging tests pass).

1. Release summary (tests, metrics, change log) is reviewed.
2. On approval, the **same** `release-<version>` image is deployed to `platform-prod`.
3. Smoke tests run against `prod.sornsub.online`.
4. If smoke tests pass, the release is marked successful.

---

## 5. Promotion Strategy

The promotion model is **build once, promote many**:

1. Build artefact in response to `main` changes.
2. Deploy and test in dev.
3. Tag as a release and deploy to staging.
4. Promote the same image to prod after manual approval.

Benefits:

- Avoids environment-specific differences in images.
- Ensures that what is tested in staging is exactly what runs in prod.
- Simplifies rollback: reverting `prod` to a previous release is simply selecting an earlier image tag and Helm revision.

---

## 6. Rollback Strategy

Rollback is handled at the deployment layer, not by rebuilding images.

### 6.1 Application Rollback

Options:

- **Helm rollback**:
  - Use Helm’s built-in revision history to roll back to a known-good release.
- **Re-deploy previous tag**:
  - Re-run deployment with the last stable `release-<version>` image.

Typical triggers for rollback:

- Smoke tests fail after deploy.
- Key health or performance metrics regress significantly.
- Critical functional defect discovered shortly after release.

### 6.2 Database Rollback

Database rollback is more complex and should be designed carefully:

- Prefer **backwards-compatible migrations**:
  - Add new columns, tables, or indexes without breaking existing code.
  - Defer destructive changes (column drops) to later deployments when safe.
- Maintain **migration version history** (e.g. Flyway or Liquibase).
- For high-risk structural changes:
  - Take a snapshot or backup before applying migrations.
  - Plan a tested rollback path (e.g. restore from snapshot in a controlled manner).

In production, rolling back the app without rolling back schema is preferred, when migrations are backwards compatible. Full DB rollback (restores) is reserved for severe incidents and follows the backup and recovery process.

---

## 7. Database Migration Strategy

Migrations are executed as part of the deployment process, with stricter controls in production.

| Environment | Migration Approach                        |
|------------|--------------------------------------------|
| Dev        | Auto-run migrations on deploy              |
| Staging    | Auto-run migrations on deploy + validation |
| Prod       | Separate migration Job, gated by approval  |

Process:

1. New migrations are committed alongside application changes.
2. Dev deploy runs all pending migrations automatically.
3. Staging deploy runs the same migrations and validates:
   - Application start-up.
   - Key read/write paths.
4. For prod:
   - A migration Job runs with the same migration set already tested in staging.
   - Once migrations succeed, the new application version is deployed.

This approach reduces the risk of untested schema changes in production.

---

## 8. Why Production Should Use the Same Image Tested in Staging

Using the same container image for staging and production is a core principle of this design.

Reasons:

- **Eliminates drift**: No risk of compiling with different flags, dependencies, or build-time environment variables between staging and prod.
- **Reproducible debugging**: Issues found in production can be reproduced using the exact same image in a lower environment.
- **Simplified compliance and audit**: It is clear which version was tested and which version is running.
- **Faster rollbacks**: Rolling back is simply switching back to a previously known-good image, not re-running builds.

In this model, only environment configuration (values files, secrets, scale parameters) differs across environments, not the binary artefact itself.
