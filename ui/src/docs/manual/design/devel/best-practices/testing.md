---
title: Testing
order: 45
---

The Taskcluster project aims for high-quality, reliable software that welcomes new contributors.
Our approach to testing has these aims:

* A consistent approach to testing across Taskcluster repositories, with a common set of supporting tools
* Repositories that are approachable for new contributors
* Confidence that production code has passed *all* tests

In particular, tests should run successfully for anyone after cloning a Taskcluster repository, and automation should ensure that code merged to upstream repos is fully tested.

The following best practices relate to Javascript applications and libraries, but apply in spirit to other languages as well.

# General Guidelines

## Approachability

Projects should be easy for new contributors to get started on quickly and independently.
Include a section in the repository's `README.md` describing how to get started modifying the code.
Services with a lockfile should suggest that contributors run `yarn install --frozen-lockfile`, while libraries should use `yarn`; after that, run `yarn test`.
The test run should be successful, even if it skips many of the tests.

## Hard Dependencies

Some services and libraries depend completely on external services, and testing without that service makes no sense.
For example, azure libraries naturally require an Azure account to test against; and pulse libraries will naturally require an AMQP server.
Similarly, services with database backends cannot be realistically tested without access to a database server.

In such cases, it is OK for the tests to refuse to run at all without configuration for the external services.
Instead, `yarn test` should fail immediately with a clear message suggesting how to set up access.
In some cases, this can be accomplished with a `docker` command (e.g., to run RabbitMQ or Postgres); others might require a free-tier account at a hosted service.

## Getting Secrets

For tests that require credentials, it should be possible to supply them in a `user-config.yml` file.
The `typed-env-config` library makes this easy.

The service repository should have a `user-config-example.yml` which has all the necessary settings filled with an illustrative example value or the string '...'.
This helps people to know which credentials they need and how to set them up.
The `user-config.yml` should be included in `.gitignore` to avoid checking in credentials.

Team members and dedicated contributors will want to know how to set up a full set of credentials for themselves.
Include a section in the README describing how to find or generate credentials for each service.

For Taskcluster credentials, include the necessary credentials in a role named `project:taskcluster:tests:<projectName>` (e.g., `project:taskcluster:tests:taskcluster-queue`).
Then instruct users how to get those scopes:

> To run *all* tests, you will need appropriate Taskcluster credentials.
> Using [taskcluster-cli](https://github.com/taskcluster/taskcluster-cli), run `eval $(taskcluster signin --scopes assume:project:taskcluster:tests:taskcluster-secrets)`, then run `yarn test` again.

# Writing Tests

Use Mocha, installed as a dev dependency, to run unit tests.

Include the following in `package.json`:

```json
"scripts": {
  "test": "mocha test/*_test.js"
}
```
(add additional patterns if tests are also located in subdirectories).

## Mocha Configuration

Include the following in `test/mocha.opts`:

```
--ui tdd
--timeout 30s
--reporter spec
```

This configures the "TDD" mocha flavor.
For reference, this defines the following helper functions:

* `suite` -- define a group of tests
* `suiteSetup` -- setup a suite (called once before any tests in the suite runs)
* `setup` -- setup each test in the suite (called once before each test in the suite runs)
* `teardown` -- tear down each test
* `suiteTeardown` -- tear down the suite

Do not use `before` or `after` -- they work unreliably with the TDD flavor.

## Test Organization

Name the test files `test/*_test.js`, so they will be matched by the `yarn test` script given below.
Naming of test scripts is at your discretion, but a common pattern is to name the tests for code in `src/foo.js`, `test/foo_test.js`.
Test files should require production code from `../src/`: `const foo = require('../src/foo')`.

Import all required modules at the top of a test file, then begin defining suites of tests.
For all `suite` and `test` calls, use the `function() { .. }` notation instead of an arrow function.
Doing so allows use of `this` to access the Mocha object.

```javascript
const frobs = require('../src/frobs.js');

suite('frobs', function() {
  test('frobnicates', async function() {
    // ...
  });
});
```

## Helpers

Include any shared test-specific code in `test/helpers.js`.

## Debug Output

A `yarn test` without any other options should produce a clean mocha report with lots of checkmarks (and `-` for skips).
To include logging output in tests, use the `debug` module so that the output is hidden by default.
When a dependent library unconditionally writes output to stdout, it's OK to leave that output interspersed with the mocha output.

## Secrets, Mocks, and External Services

Test cases that require access to an external service should also be able to run against a mock version of that external service, and should run twice, once with the real service, and once with the mock implementation.
When the service credentials are not available, the "real" condition should be skipped.
If the environment variable `NO_TEST_SKIP` is set, lack of credentials should be treated as an error.

### Storing and Retrieving Secrets

Use the `taskcluster-lib-testing` `Secrets` class to handle retrieving secrets during test runs.
It takes a secret name, pointing to a secret defining a set of environment variables.
That secret should be named `project/taskcluster/testing/<projectName>`.

This utility handles all of the details of running tests twice (with and without mocks), fetching secrets from the secret service in CI, skipping when secrets are not available, and failing when `$NO_TEST_SKIP` is set.

### Expensive Services

Some external dependencies are too expensive or complex to test against.
For example, the AWS EC2 APIs are difficult to use in testing, and problems with tests could easily become very expensive.
In such cases, it's OK to only test against mocks.

In this case, use something like the following in your test suite, using (mock) to indicate that these are against a mock.
There are no tests against any "real" service, so there's nothing to skip.

```javascript
suite('spawning EC2 instances (mock)', function() {
  // ...
});
```

### Available Mock Implementations

The following is a partial list of useful mock implementations.

* `fakeauth` in `taskcluster-lib-testing` provides a fake implementation of the Auth service for testing services that define APIs.
* The `Entity` class in `azure-entities` can take an `inMemory: true` option to store data locally without requiring Azure credentials.
* The `aws-mock-s3` package provides good support for mocking the `S3` component of the AWS SDK.
* The `nock` package is useful for intercepting HTTP requests very low in the node HTTP client implementation, and is especially helpful when mocking other TC services.
  For example, `fakeauth` uses `nock`.

Development of new, general-purpose mocks is encouraged!

## Test Output

In the default configuration, where possible, tests should output only the pass/fail status of each test case (Mocha's default output).
Any additional diagnostic output should be handled with debug logging or otherwise disabled by default.

# CI Configuration

Tests should run in Taskcluster itself, via Taskcluster-Github.

Pushes to the upstream repository should ensure that *all* tests run, by setting `NO_TEST_SKIP`.
In cases where tests require credentials, but they are sufficiently harmless that they can be provided to unreviewed pull requests, configure `.taskcluster.yml` to also set `NO_TEST_SKIP` for pull requests.
For example, pulse credentials are trivial for anyone to acquire and easily revoked, so exposing a set of testing pulse credentials does no harm.

Where necessary, provide the appropriate `secrets:get:` scope in the `repo:github.com/taskcluster/taskcluster-lib-foo:branch:master` and (if harmless) `repo:github.com/taskcluster/taskcluster-lib-foo:pull-request` roles.

# ESLint

Use [eslint-config-taskcluster](https://github.com/taskcluster/eslint-config-taskcluster) to check for lint.

To do so, install `eslint-config-taskcluster` as a dev dependency and make a scripts section of the package.json that is similar to

```json
"scripts": {
  "lint": "eslint src/*.js test/*.js",
  "pretest": "yarn lint"
},
```
(again, add more patterns if necessary, especially if `src` or `test` contain subdirectories)

Create `.eslintrc` in the root of the repository with
```js
{
  "extends": "eslint-config-taskcluster"
}
```
