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
const appendSuffix = (text: string): string => text.concat("IAMASUFFIX");
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

Integration tests follow many of the same principles as unit tests except rather than test units of code it is the access points that are tested, they should still be short, isolated, idempotent, and readable. Integration tests running with mocked servers should be fast to run and are a good test strategy for micro services which have their `E2E` tests in a parent repo.

While good unit tests reduce the need for integration tests their value should not be dismissed, they give a level of confidence in the system that all of the units communicate as expected. Integration tests are typically a lot fewer in number that the unit tests and run fairly quickly, ideally on `PRs` and merges to the main branch.

## End to End Tests (E2E)

End to end tests are very high level and idealy replicate the actions of a user through the system, for example In the case of `Reviewer` this means browser testing and the user's interactions with the webpage is replaced with functions that click on the apropriate elements and type on the apropriate inputs. E2E tests can take many forms but this document will focus mainly on browser tests as that is the most common ones that this team has.

Browser tests like all others should be isolated, idempotent, and should only mock things that must be mocked, this does not mean that the tests run agains the production systems but they should run on something that is identical (but with fake data in a lot of cases). Care needs to be taken to ensure that if test A fails that it doesn not prevent test B from running, a good example of this would be something that stores a value object in a datastore. While testing that this happens successfuly is fine, subsequent tests should not rely on that piece of data being stored.

Having a repository if mocked / tests data that can be used to quickly bring up an instance of a database (where one is needed) is invaluable when writing E2E tests. This allows us to carefully shape the data we test to include as many real life examples as we need without the weight of a complete production dataset, it also allows us to easilly add data for regression tests without poluting the production systems.

E2E tests are often slow and require expensive setups like opening a new browser instance or prepopulating a database. It is great to have a wide range of these tests but caution should be observed of the time it takes to run, especially on `CI`. The browser tests on reviewer as of the time of writing this take several hours to run across the 4 browsers. When determining what E2E tests to write it is useful to look at the user stories. For example given a user story of "As a researcher I would like to give a quick indicator of my aproval for a preprint" in the `EPP` project we could write a test that replicated a user logging in, navigating to that document, clicking a thumbs up button and then assert that the count displayed for that has gone up.

When writing browser tests helper functions are usually created to shorten and clarify the test code. In the `Reviewer` and `Editor` projects we used page objects, these are a collection of functions that represent a page of the application, for example in the `Reviewer` project we had one for each step of the form and each one had: a function to auto fill in all or a minimum set of fields, a function to get the status of various elements, and one to set the value of each input box. There was also a navigation helper that would allow the developer to do the following:

```ts
const navigationHelper = new NavigationHelper();
await navigationHelper.navigateToDetailsPage(true, "Feature Article");
```

This created an instance of the navigation helper and then uses the test framework to enact the minimum happy path to get to the details page setting all of the fields to be complete and the article type to be a 'Feature Article'. The other big advantage of this is a single place in the tests where selectors are defined, if an input box has its class changed then it only needs to be changed on relevant page object.

## Mocking

## Debugging

## Test Frameworks

## When to test

## Pitfalls
