# Testing Principles
This page is a series of principles that the team tries to follow when testing our code. It is important to note that these are guidelines to be followed where possible and not hard rules.

## What to test
What we test and what we do not test are very important, testing the wrong things leads to over-testing and slow CI and if we miss tests on important areas we may miss issues before they are found in production.

These are the things we should be aiming to test:
 - Every unit of code should have a corresponding test (Unit Tests).
 - All the interactions between our systems should be tested (Integration Tests).
 - All the user's interactions with the application should be tested (E2E/Browser Tests).
 - The rendered pages should be tested for visual regressions (Snapshot Tests).

This is what we should not be testing:
 - Third party code. It is not up to us to ensure that these dependencies are tested, this will be handled in E2E/Browser tests.
 - Language features. There is very little to be gained from testing the following function directly when it will be tested where it is used in other units;
```ts
const appendSuffix = (text: string): string => text.concat('IAMASUFFIX');
```
Deciding what to test when it comes to the slower E2E/Browser tests is complicated. In an ideal world every little interaction would be tested as a matter of course, however these tests are often slow to run and can make the developer experience frustrating when waiting hours for a CI run to finish. This problem can be approached by:
 - Testing the requirements. This assumes that there are a hard set of requirements documented somewhere that can be used to generate a test list. This should be all the tests, but it is a good indicator of the minimum.
 - Where possible it can be very useful to group slower tests so that they can be run in different scenarios. In the past the following groups have proven effective:
   - all: All of the tests. Run on merges to the main branch or before a deployment.
   - core: The core happy path tests that follow select user journeys.
   - slow: Some tests are just slow by design or as a consequence of what is being tested, it is often more convenient to run these only on `ci` or before a push when developing.

## Unit Tests
Unit tests are aimed at testing isolated components of code, these are often the smallest accessible building blocks of the application. What can actually be tested depends somewhat on the language and testing frameworks being used. For example, in `Java` it is possible to test private functions via reflection whereas in `TypeScript` it is not.

In a language like `TypeScript` each function can be considered to be a unit, however this means that there are problems testing things that are not exported. This can easily fixed by exporting all functions from the file but this should be avoided as it will only make the codebase more confusing and there are functions that you may not want to expose. These units that are not exposed should be being used by the ones that are (if they are not then they can be deleted), this means that to attain good test coverage the tests for the exposed functions should take these into account.

For a unit to be considered tested all of the branches of the unit should be tested, commonly by calling the function in different ways for each test, for example: if there is a `switch` in the function then all of the `cases` should be covered by a test (within reason). If the function calls out to other private functions within the file that contain things like `if` or `switch` statements then these should be taken into account when writing the test for the main unit.

Unit tests should be specific, short, isolated, idempotent, and readable:
  - specific: Each test should be clearly defined and test a unique case, for example: a test for a function that branches early on in an `if` statement should only cover one branch per test.
  - short: A unit test can sometimes have a fair amount of code to set it up but the actual test wants to be a short and clear as possible. Preferably `some setup -> call function -> assert on result`. Tests ideally can often be less than 10 lines of code.
  - isolated: Not exclusive to unit tests but no test should depend on the result of another test, this just introduces points of failure and can lead to very frustrating debugging of tests to isolate issues where one is dependent on another.
  - idempotent: Tests that are run should always return the same result. No matter how many times they are run if a test passes or fails, without changing the test or underlying code then that result should prevail. Inconsistent tests can be one of the most frustrating things for a developer to deal with.
  - readable: Tests can be used as a living documentation of the code. Frameworks like `Jest` lend themselves to this incredibly well and allow for good separation of concerns and descriptors for each test. In the absence of or with poor quality documentation tests are one of the quickest ways to work out how and what the code does.

## Integration Tests
These test the communication points between parts of the application. For example: a back end application running an `api` can be tested by making requests to the `api` like any other application would. In this instance any external services would be mocked like databases or external servers. This is very useful in `client/server` systems like `Reviewer` as it allows testing and development to happen independently when a contract on the endpoints has been made with a high level of confidence that the communication will work as expected. Note: this does not mean that the systems should not be tested together (see `E2E` tests).

Integration tests follow many of the same principles as unit tests except rather than test units of code it is the access points that are testes, they should still be short, isolated, idempotent, and readable. Integration tests running with mocked servers should be fast to run and are a good test strategy for micro services which have their `E2E` tests in the parent repo.

While good unit tests reduce the need for integration tests their value should not be dismissed, they give a level of confidence in the system that all of the units communicate as expected. Integration tests are typically a lot fewer in number that the unit tests and run fairly quickly, ideally on `PR`s and merged to the main branch.
