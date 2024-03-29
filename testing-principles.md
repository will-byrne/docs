# Testing Principles

This page is a series of principles that the team tries to follow when testing our code. It is important to note that these are guidelines to be followed where possible and not hard rules.

## What to test

What we test and what we do not test are very important, testing the wrong things leads to over-testing and slow CI and if we miss tests on important areas we may miss issues before they are found in production.

These are the things we should be aiming to test:

- Every unit of code should have a corresponding test (Unit Tests).
- All the interactions between our units should be tested (Integration Tests).
- All the user's interactions with the application should be tested (E2E/Browser Tests).
- The rendered pages should be tested for visual regressions (Snapshot Tests).

This is what we should not be testing:

- Third party code. It is not up to us to ensure that these dependencies are tested, this will be indirectly tested by E2E/Browser tests.
- Language features. There is very little to be gained from testing the following function directly when it will be tested where it is used in other units;

```ts
const appendSuffix = (text: string): string => text.concat("IAMASUFFIX");
```

Deciding what to test when it comes to the slower E2E/Browser tests is complicated. In an ideal world every little interaction would be tested as a matter of course, however these tests are often slow to run and can make the developer experience frustrating when waiting hours for a CI run to finish. This problem can be approached by:

- Testing the requirements. This assumes that there are a hard set of requirements documented somewhere that can be used to generate a test list. This will almost certainly not be all the tests, but it is a good indicator of the minimum.
- Where possible it can be very useful to group slower tests so that they can be run in different scenarios. In the past the following groups have proven effective:
  - all: All the tests. Run on merges to the main branch or before a deployment.
  - core: The core happy path tests that follow select user journeys.
  - slow: Some tests are just slow by design or as a consequence of what is being tested, it is often more convenient to run these only on `ci` or before a push when developing.

## Unit Tests

Unit tests are aimed at testing isolated components of code, these are often the smallest accessible building blocks of the application. What can actually be tested depends somewhat on the language and testing frameworks being used. For example, in `Java` it is possible to test private functions via reflection whereas in some other languages it is not.

In a language like `TypeScript` each function can be considered to be a unit, however this means that there are problems testing things that are not exported. This can easily be fixed by exporting all functions from the file but this should be avoided as it will only make the codebase more confusing and there are functions that you may not want to expose. These units that are not exposed should be being used by the ones that are (if they are not then they can be deleted), this means that to attain good test coverage the tests for the exposed functions should take these into account.

For a unit to be considered tested all the branches of the unit should be tested, commonly by calling the function in different ways for each test, for example: if there is a `switch` in the function then all of the `cases` should be covered by a test (within reason). If the function calls out to other private functions within the file that contain things like `if` or `switch` statements then these should be taken into account when writing the test for the main unit.

Unit tests should be specific, short, isolated, idempotent, and readable:

- specific: Each test should be clearly defined and test a unique case, for example: a test for a function that branches early on in an `if` statement should only cover one branch per test.
- short: A unit test can sometimes have a fair amount of code to set it up but the actual test wants to be a short and clear as possible. Preferably `some setup -> call function -> assert on result`. Tests ideally can often be less than 10 lines of code.
- isolated: Not exclusive to unit tests but no test should depend on the result of another test, this just introduces points of failure and can lead to very frustrating debugging of tests to isolate issues where one is dependent on another.
- idempotent: Tests that are run should always return the same result. No matter how many times they are run if a test passes or fails, without changing the test or underlying code then that result should prevail. Inconsistent tests can be one of the most annoying things for a developer to deal with and often leads to people ignoring test results.
- readable: Tests can be used as a living documentation of the code. Frameworks like `Jest` lend themselves to this incredibly well and allow for good separation of concerns and descriptors for each test. In the absence of or with poor quality documentation tests are one of the quickest ways to work out how and what the code does.

## Integration Tests

These test the communication points between parts of the application. For example: a back end application running an `api` can be tested by making requests to the `api` like any other application would. In this instance any external services would be mocked like databases or external servers. This is very useful in `client/server` systems like `Reviewer` as it allows testing and development to happen independently when a contract on the endpoints has been made with a high level of confidence that the communication will work as expected. Note: this does not mean that the systems should not be tested together (see `E2E` tests).

Integration tests follow many of the same principles as unit tests except rather than test units of code it is the access points that are tested, they should still be short, isolated, idempotent, and readable. Integration tests running with mocked servers should be fast to run and are a good test strategy for microservices which have their `E2E` tests in a parent repo.

While good unit tests reduce the need for integration tests their value should not be dismissed, they give a level of confidence in the system that all the units communicate as expected. There are typically fewer Integration tests than unit tests, and they run fairly quickly, ideally on `PRs` and merges to the main branch.

## End-to-End Tests (E2E)

End-to-end tests are very high level and ideally replicate the actions of a user through the system, for example In the case of `Reviewer` this means browser testing and the user's interactions with the webpage is replaced with functions that click on the appropriate elements and type on the appropriate inputs. E2E tests can take many forms but this document will focus mainly on browser tests as that is the most common ones that this team has.

Browser tests like all others should be isolated, idempotent, but should only mock things that must be mocked, this does not mean that the tests run against the production systems, but they should run on something that is identical (but with fake data in a lot of cases). Care needs to be taken to ensure that if test A fails that it does not prevent test B from running, a good example of this would be something that stores a value object in a datastore. While testing that this happens successfully is fine, subsequent tests should not rely on that piece of data being stored.

Having a repository of mocked / tests data that can be used to quickly bring up an instance of a database (where one is needed) is invaluable when writing E2E tests. This allows us to carefully shape the data we test to include as many real life examples as we need without the weight of a complete production dataset, it also allows us to easily add data for regression tests without polluting the production systems.

E2E tests are often slow and require expensive setups like opening a new browser instance or repopulating a database. It is great to have a wide range of these tests but caution should be observed of the time it takes to run, especially on `CI`. The browser tests on reviewer as of the time of writing this take several hours to run across the 4 browsers.

When determining what E2E tests to write it is useful to look at the user stories. For example given a user story of "As a researcher I would like to give a quick indicator of my approval for a preprint" in the `EPP` project we could write a test that replicated a user logging in, navigating to that document, clicking a thumbs up button and then assert that the count displayed for that has gone up.

When writing browser tests, helper functions are usually created to shorten and clarify the test code. In the `Reviewer` and `Editor` projects we used page objects, these are a collection of functions that represent a page of the application, for example in the `Reviewer` project we had one for each step of the form and each one had: a function to autofill in all or a minimum set of fields, a function to get the status of various elements, and one to set the value of each input box. There was also a navigation helper that would allow the developer to do the following:

```ts
const navigationHelper = new NavigationHelper();
await navigationHelper.navigateToDetailsPage(true, "Feature Article");
```

This created an instance of the navigation helper and then uses the test framework to enact the minimum happy path to get to the details page setting all the fields to be complete and the article type to be a 'Feature Article'. The other big advantage of this is a single place in the tests where selectors are defined, if an input box has its class changed then it only needs to be changed on relevant page object.

## Mocking
Mocking is where a dependency is replaced with something under the testers control, this can be a stub, a replacement function or just a no-op to keep the type system happy. Mocking allows for testing only the local code without caring whether anything else is working as expected, it adds a granularity to the tests so that when `dependency a` is broken you can still run your tests.

for example:

```ts
import fetch from 'node-fetch';

// a function that retrieves an image from the a server and adds it to a datastore returning true if successful
export const getImage = async (url, datastore): Promise<boolean> => {
  const imageData = await fetch(url, { headers: 'some_header'});
  const success = await datastore.store(imageData.buffer, imageData.name, imageData.encoding);
  return success;
};
```
This is a simple function that retrieves an image and stores it in a datastore with the name and the encoding. To unit test this in isolation the `fetch` call and the `datastore.store` call should both be mocked so the tests are not in any way dependent on services outside the unit. In jest this would be done simply with:
```ts
import { getImage } from './image-processing';
import fetch from 'node-fetch';
const fetchMock = jest.mock('node-fetch');

describe('test', () => {
  it('stores the image with the image name and correct encoding', async () => {
    const datastoreMock = {
      store: jest.fn().mockImplementation(() => Promise.resolve(true)),
    };
    fetchMock.mockResolve({ buffer: 'some mocked buffer', name: 'i am a mock image', encoding: 'base64'});

    const result = await getImage('test-url', datastoreMock);

    expect(datastoreMock.store).toHaveBeenCalledWith('some mocked buffer', 'i am a mock imnage', 'base64');
  });
});
```
What this is doing is creating a stubbed mock of the import of `fetch` which is then being told to mock a resolved promise with an object we have defined. The datastore only had one function being used, so we are telling jest that it should return a resolved promise of `true`. Finally, in the `expect` we are asserting the datastore mocked function was called with the specified parameters.
> note: this is a simple example and mocking can be used for much more complex cases.

### What to mock
When writing unit tests in an ideal world everything outside the unit being tested should be a mock. Realistically this leads to slower test writing and more complex tests, how much is mocked is something that should be agreed on by the team for each project. At a minimum it is recommended that everything external to the codebase especially everything that is an async call should be mocked. This is to ensure that no external code is being tested and our tests don't depend on the status and quality of external services.

For integration, end to end, and browser tests all external services should be mocked as per the unit tests. External calls here can be mocked either internally or by using a mock service that only returns what is set beforehand and can be created / destroyed per test run. For example if the application uses Amazon S3 buckets then in the integration and end-to-end tests a docker container can be started  like S3mock which is preloaded with the data the tests require.

### Types of mock
- Stub: an object matching the shape of the item being mocked with no-op functions replacing real ones, great for when you don't care about its return value in the test.
- Function: a function that matches the types of the one being mocked that will always return a hardcoded value for use in the test.
- Hardcoded data: Often used in conjunction with the function mock, this is a store result or object that can be used in the test.
- Services: a clone of an external service that matches the shape of its api. Easy to manage with docker and to bring up a fresh instance with the dataset required for the test suite.
- Spies: almost synonymous with mock but spies are used to examine the state of a mock that is being used in a test, in the above example the jest mock for `datastore.store()` allows assertions on what arguments the function was called with.

## Debugging
Debugging tests is the same as any other application. Using an IDE such as Webstorm allows line by line inspection as does various node terminal runners. Simply using `console.log` in test is often enough to see what is being called and what the state of various stages of the code is at when the test is running.
> note: remember to remove testing logs from the codebase before pushing back to source control to avoid polluting the application logs.

## Test Frameworks
Test frameworks are used to help write, architect and run your tests. There are many frameworks per language and the choice of which to use is a team decision, for example [Testing-library](https://testing-library.com/) is a very useful tool for testing web based front end units in `js` / `ts` especially React, Vue, Angular, etc....

### Advantages
Test frameworks can take a lot of the burden and boilerplate out of writing tests. [Testing-library](https://testing-library.com/) allows for individual React components to be rendered to a virtual dom simulating the browser environment without needing to worry about using something like PhantomJS, it also simulates the user interaction with the component as if it were in a browser.

Frameworks such as [Playwright](https://playwright.dev/) will connect to a running web front end and allow for user interaction and testing to be done on code running in the browser

### Disadvantages
Many test frameworks are very opinionated about how the tests should be written, this is fine as long as it suits the way the team wants to test but can cause issues with complexity and make the tests harder to write.

Test frameworks can also require a lot of time to learn to use effectively. This is particularly noticeable when writing browser tests as they require some way of targeting the different elements on the page and these are usually different.
## When to test
Testing should be started before even the first line of code is written (where possible). This is known as test driven development and as long as you know the shape of your functions or have a reasonable approximation of their output then you can write the tests ahead of time and code until the tests pass. This is an area where having strong requirements can really help as they can be used to derive the test cases which can help when deciding how to design the project. Regression tests should be written as soon as work starts to fix a bug, even if these tests do not remain in the code base they are often very useful in determining the cause of the issue and indicating when it is fixed.

## Common mistakes
### Over / Under testing
When writing tests getting the correct amount of coverage without going overboard is difficult. It is a function of coverage / time as to how many tests you can write on the system. Part of this is down to the resources available wherever the tests are being run and even the quality of the tests and frameworks used. The longer tests take to run the more of a chore they seem to be and the more they increase the maintenance burden.

### Complex tests
No matter what kind of tests you are writing, they actual test code should be as simple as possible. Simple tests are more maintainable and easy to read making them more useful as code documentation and quicker to alter and add to.

It is usually possible to separate complex test code to helper functions. For example: in  browser tests using page objects to hide the complex navigation, in unit tests using a helper function to create a complex object or mock, or code for starting and stopping mock services for end-to-end tests.
