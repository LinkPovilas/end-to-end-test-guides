# Introduction

This document outlines the conventions employed when writing end-to-end (E2E) tests. The code examples predominantly utilize TypeScript and Playwright syntax.

# Table Of Contents

- [Introduction](#introduction)
- [Table Of Contents](#table-of-contents)
- [Web Elements](#web-elements)
  - [Selectors](#selectors)
  - [Locators](#locators)
- [Popular Test Design Patterns](#popular-test-design-patterns)
  - [Page Object Model (POM)](#page-object-model-pom)
  - [Cucumber (BDD)](#cucumber-bdd)
  - [Screenplay](#screenplay)

# Web Elements

An element is a component of a webpage. In XML and HTML, an element may contain a data item, a chunk of text, an image, or even nothing at all. A typical element consists of an opening tag with some attributes, enclosed text content, and a closing tag.

## Selectors

Selectors are strings or patterns used to identify and select elements within the DOM (Document Object Model) of a webpage.

Types of selectors:

- CSS (Cascading Style Sheets) selectors - patterns used to select elements to which a set of CSS rules are then applied.

  - `#id`
  - `.class`
  - `div > p`
  - `[type="submit"]`
  - `<...>`

- XPath (XML Path Language) - a query language for selecting nodes from an XML-like document.
  - `//div[@id='content']`
  - `//button[text()='Submit']`
  - `<...>`

The CSS selector strategy is preferred for its simplicity, speed, and ease of use. Most front-end developers are already familiar with CSS selectors because they use them extensively for styling purposes. Additionally, web browsers are highly optimized for CSS selectors since they are heavily used in rendering web pages.

## Locators

Locators are a higher-level abstraction in testing frameworks (such as Selenium, Playwright, and Cypress) used to find elements for interaction or verification in test scripts. Selectors are employed as part of their strategy to locate elements.

```javascript
// HTML element
<button
  id="submit-button"
  class="btn-primary"
  name="submit"
  type="submit"
  data-testid="submit-button"
>
  Submit
</button>;

// Selenium
driver.findElement(By.cssSelector("[data-testid='submit-button']"));

// Playwright
page.getByTestId("submit-button");

// Cypress
cy.get('[data-testid="submit-button"]');
```

# Popular Test Design Patterns

Below are the test design patterns commonly used in web automation.

## Page Object Model (POM)

Webpages are represented as classes, and the various elements on the page are defined as class properties. User interactions are implemented as class methods. The goal of using page objects is to abstract any page information away from the actual tests.

The traditional Page Object Model (POM) assumes static web pages where each page corresponds to a single URL. However, modern web applications often use dynamic components that load asynchronously or are reused across different pages. POM struggles to handle such component-based architectures because it focuses on pages as discrete entities. When components change or new ones are added, maintaining the corresponding POM classes becomes cumbersome. You would need to update multiple classes across different pages.

> [!IMPORTANT]
> See [style guide](/docs/playwright-style-guide.md) for a better POM implementation in the form of Lean Page Object pattern.

```typescript
// ./pages/login-page.ts
export class LoginPage {
  private readonly page: Page;
  readonly emailField: Locator;
  readonly passwordField: Locator;
  readonly loginButton: Locator;
  readonly searchField: Locator;
  readonly searchButton: Locator;
  readonly profileLink: Locator;
  readonly logoutButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailField = page.getByTestId("email");
    this.passwordField = page.getByTestId("password");
    this.loginButton = page.getByTestId("submit-button");
    this.searchField = page.getByTestId("search");
    this.searchButton = page.getByTestId("search-button");
    this.profileLink = page.getByTestId("profile-link");
    this.logoutButton = page.getByTestId("logout-button");
  }

  async navigateToLoginPage() {
    await this.page.goto("/");
  }

  async enterEmail(email: string) {
    await this.emailField.fill(email);
  }

  async enterPassword(password: string) {
    await this.passwordField.fill(password);
  }

  async clickLoginButton() {
    await this.loginButton.click();
  }

  async login(email: string, password: string) {
    await this.enterEmail(email);
    await this.enterPassword(password);
    await this.clickLoginButton();
  }

  async searchForItem(item: string) {
    await this.searchField.fill(item);
    await this.searchButton.click();
  }

  async navigateToProfile() {
    await this.profileLink.click();
  }

  async logout() {
    await this.logoutButton.click();
  }
}

// ./tests/authentication.spec.ts
/** test function removed for brevity */
const loginPage = new LoginPage(page);
await loginPage.login("test@company-mail.com", "TestPassword123$");
```

## Cucumber (BDD)

Cucumber is a widely used tool for Behaviour Driven Development (BDD). Plain text feature files are created to describe the expected behaviour of a feature as scenarios written in Gherkin syntax. Steps in the feature files are bound to step definitions, which are wrapper functions where the actual code is executed.

It’s primarily a collaboration tool, not a testing tool. It requires external test libraries and custom setup for writing tests. When Cucumber is adopted solely as a tool to write automated tests without any input from business analysts, it becomes inappropriate. Additionally, the test runner lacks critical features, such as the ability to split and run scenarios on several separate machines.

```gherkin
# ./features/authentication.feature
Feature: Authentication

  Background:
    Given I navigate to the landing page

  Scenario: Valid user login
    When I login as "valid_user"
    Then I should be redirected to the contact list page
```

```javascript
// ./features/step-definitions/authentication.steps.ts
Given("I navigate to the landing page", async function (this: CustomWorld) {
  await goToLandingPage(this.page);
});

When(
  "I login as {string}",
  async function (this: CustomWorld, userType: string) {
    await loginAs(this.page, userType);
  }
);

Then(
  "I should be redirected to the contact list page",
  async function (this: CustomWorld) {
    await ensureUrl(this.page, /\/contactList/);
  }
);

// src/helper-methods/authentication.ts
export const enterEmail = async (page: Page, email: string) => {
  await page.getByPlaceholder("Email").fill(email);
};

export const enterPassword = async (page: Page, password: string) => {
  await page.getByPlaceholder("Password").fill(password);
};

export const clickSubmit = async (page: Page) => {
  await page.getByRole("button", { name: "Submit" }).click();
};

export const loginAs = async (page: Page, userType: string) => {
  const { email, password } = getUserData(userType);
  await enterEmail(page, email);
  await enterPassword(page, password);
  await clickSubmit(page);
};

// ./src/helper-methods/navigation.ts
export const goToLandingPage = async (page: Page) => {
  await page.goto("/");
};

export const ensureUrl = async (page: Page, urlOrRegExp: string | RegExp) => {
  await expect(page).toHaveURL(urlOrRegExp);
};
```

## Screenplay

It is a command pattern where `Actor` objects utilize `Abilities` to execute command objects. This approach encourages good programming habits and emphasizes writing code that is easy to understand.

However, it’s not widely adopted due to its steep learning curve. Interestingly, it complements Cucumber well when implemented together. The most popular testing library provides its own wrappers for several web automation libraries, but it doesn’t expose all of the library-specific underlying APIs. As a result, you may need to create workarounds for certain cases. Additionally, this approach requires you to store state in objects because you can’t directly assign values to variables. Debugging can be somewhat awkward, and you might inadvertently introduce bugs into your tests, especially when dealing with custom union types like `Answerable<T>`.

```typescript
// ./page-objects/login-form.ts
export class LoginForm {
  static usernameField = () =>
    PageElement.located(By.css("#username")).describedAs("username field");

  static passwordField = () =>
    PageElement.located(By.css("#password")).describedAs("password field");

  static loginButton = () =>
    PageElement.located(By.css("#login-button")).describedAs("login button");
}

// ./tests/authentication.spec.ts
/** test function removed for brevity */
await actorCalled("Alice").attemptsTo(
  Navigate.to("/"),
  Ensure.that(Text.of(LoginForm.loginButton()), equals("Login")),
  Enter.theValue("valid_user").into(LoginForm.usernameField()),
  Enter.theValue("valid_password").into(LoginForm.passwordField()),
  Click.on(LoginForm.loginButton()),
  Ensure.eventually(
    Text.of(PageElement.located(By.css("h1"))),
    equals("Contact List")
  )
);
```
