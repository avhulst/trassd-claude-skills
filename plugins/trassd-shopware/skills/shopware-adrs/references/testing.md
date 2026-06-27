# Testing

## Follow the test pyramid (2023-02-13)

**Decision:** Shopware's test suite had grown into an inverted pyramid — too
many slow, flaky end-to-end and integration tests, too few unit tests. The
decision is to commit to the test pyramid: many fast unit tests at the base, a
thin layer of high-value E2E tests at the top. E2E tests that merely exercise
CRUD or admin-module behaviour are cut and replaced with Jest or PHP
integration/API tests; only tests that genuinely verify an important feature
end-to-end remain, and those are vetted thoroughly (run many times) before
merging.

**Rule for extensions:** Cover logic with unit tests and use PHP
integration/API or Jest tests for CRUD-style behaviour; reserve E2E tests for
critical user-facing flows that can't be verified any other way. Avoid building
your plugin's quality gate on a large, flaky E2E layer.

## Mocking repositories

The repository-mocking ADR (2023-04-01, `StaticEntityRepository`) supports
writing fast unit tests for repository-dependent classes — see
[extension-patterns.md](extension-patterns.md).
</content>
