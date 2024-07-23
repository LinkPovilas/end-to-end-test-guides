# Introduction

This document describes the standards used for writing E2E (end-to-end) tests using Playwright.

# Table Of Contents

- [Introduction](#introduction)
- [Table Of Contents](#table-of-contents)
- [Project Structure](#project-structure)
- [Naming Files \& Folders](#naming-files--folders)
- [Web Elements](#web-elements)
  - [Selecting Elements](#selecting-elements)
  - [Naming Page Elements](#naming-page-elements)
- [Page Objects](#page-objects)
  - [Grouping Page Elements Into Lean Page Objects](#grouping-page-elements-into-lean-page-objects)
  - [Nested Page Objects](#nested-page-objects)
  - [Using Test Fixtures For Initializing Objects](#using-test-fixtures-for-initializing-objects)
  - [Defining Test Scenarios](#defining-test-scenarios)

# Project Structure

To make navigation easier keep the project structure flat. Consider using a tool like [this website](https://tree.nathanfriend.io/) to plan your project structure.

```Shell
.
├── api          # API related data like interfaces, etc.
├── data         # Test data
├── fixtures     # Dependency injection and state handling
├── page-objects # Lean page objects
└── tests        # Test cases
```

# Naming Files & Folders

Use [kebab-case](https://developer.mozilla.org/en-US/docs/Glossary/Kebab_case) for folder and file names. Use plural case for directories (folders) and singular case for file names. For example:

- Folder: `./page-objects/`
- File: `./page-objects/page-object.ts`

Follow this formula: `<descriptor>-test.ts` when naming fixtures. For instance:

- Fixture: `./fixtures/page-object-test.ts`

# Web Elements

## Selecting Elements

Your tests should resemble how users interact with the application as much as possible. With this in mind, the recommended order of priority:

| Strategy       | Notes                                                                                                                                                                                                                                             |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| By role        | This can be used to query every element that is exposed in the [accessibility tree](https://developer.mozilla.org/en-US/docs/Glossary/Accessibility_tree). Coupled to the `name` attribute.                                                       |
| By label       | This method is really good for form fields, because it emulates the behavior of users finding elements using label text.                                                                                                                          |
| By placeholder | If form elements do not have labels but do have placeholder texts.                                                                                                                                                                                |
| By alt text    | When the element supports alt text such as `img`, `area`, `input`, and any custom element.                                                                                                                                                        |
| By title       | When your element has the title attribute. The title attribute is not consistently read by screenreaders, and is not visible by default for sighted users.                                                                                        |
| By text        | Recommend for non interactive elements (div, span, p, etc).                                                                                                                                                                                       |
| By test id     | It is recommended to use this only after the other queries don't work for your use case. Users cannot see or hear these, so this is only recommended for cases where you can't match by role or text. Attribute name `data-*` (ex:`data-testid`). |

**Examples:**

By role:

```javascript
// <label><input type="checkbox" /> Remember Me </label>
page.getByRole("checkbox", { name: "Remember Me" });
```

By label:

```javascript
// <label>Password <input type="password" /></label>
page.getByLabel("Password");
```

By placeholder:

```javascript
// <input type="email" placeholder="name@example.com" />
page.getByPlaceholder("name@example.com");
```

By alt text:

```javascript
// <img alt="company logo" src="/img/logo.svg" width="100" />
page.getByAltText("company logo");
```

By title:

```javascript
// <span title="Comment count">25 comments</span>
page.getByTitle("Comment count");
```

By text:

```javascript
// <span>Waiting for bank response...</span>
page.getByText(/waiting for bank response/i);
```

By test id:

```javascript
// <button data-testid="submit-button">Log in</button>
page.getByTestId("submit-button");
```

## Naming Page Elements

Pay attention to the semantics of HTML attributes, such as class attribute values. When naming elements use formula: `<descriptor><Type>`.

References:

- [HTML Elements](https://developer.mozilla.org/en-US/docs/Web/HTML/Element)
- [ARIA roles](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles)

Examples:

Text field:

```html
<label for="password-input">Password:</label> <input id="password-input" />
```

```javascript
const passwordField = page.getByLabel(/password/i);
```

Button:

```html
<a
  href="https://some-url.com/internet-bank"
  class="btn a btn-primary"
  target="_self"
  rel="noopener noreferrer"
  data-once="aBtn"
  ><span>Internet Bank</span></a
>
```

```javascript
const internetBankButton = page.getByRole("link", { name: "Internet Bank" });
```

Dropdown menu and it's item:

```html
<li
  class="main-menu-item nav-item pl-lg-3 pr-lg-3 dropdown menu-item__right menu-item-3"
  data-top-item-id="menu_link_content"
  data-once="fix-menu-position"
>
  <a
    class="nav-link ml-lg-n1 mr-lg-n1 u-h96--lg dropdown-toggle text-wrap show"
    role="button"
    data-bs-toggle="dropdown"
    aria-haspopup="true"
    aria-expanded="true"
    title="Career"
    data-once="headerMenuLevel0A11y navLinkSpace"
    >Career</a
  >
  <ul
    class="dropdown-menu show"
    style="left: -321.567px; width: 483px;"
    data-bs-popper="static"
  >
    <li class=" ">
      <a
        href="/en/career"
        class="dropdown-item "
        title="Career"
        data-once="drodownItemSpace"
        >Career</a
      >
    </li>
  </ul>
</li>
```

```javascript
const careerDropdownMenuButton = page.getByRole("button", { name: "Career" });
const careerLink = page.getByRole("link", { name: "Career" });
```

# Page Objects

## Grouping Page Elements Into Lean Page Objects

- Use minimalistic page objects that are focused only on grouping related page elements.
- Use getter functions, because they are evaluated when you access the property, not when you generate the object. With that you always request the element before you do an action on it. This can help handle application state changes more effectively and reduce test flakiness.
- A page object should only contain interactions that are available with that component.

```typescript
// page-object.ts
abstract class PageObject {
  constructor(protected readonly page: Page) {}
}

// login-form.ts
class LoginForm extends PageObject {
  get emailField() {
    return this.page.getByLabel("Email");
  }

  get passwordField() {
    return this.page.getByLabel("Password");
  }

  get submitButton() {
    return this.page.getByRole("button", { name: "Submit" });
  }

  async enterEmail(email: string) {
    await this.emailField.fill(email);
  }

  async enterPassword(password: string) {
    await this.passwordField.fill(password);
  }

  async clickSubmit() {
    await this.submitButton.click();
  }

  async login(email: string, password: string) {
    await this.enterEmail(email);
    await this.enterPassword(password);
    await this.clickLoginButton();
  }
}
```

## Nested Page Objects

When you have components inside of other components you need to handle it accordingly.

```typescript
// career-dropdown-menu.ts
export class CareerDropdownMenu extends PageObject {
  get self() {
    return this.page.getByRole("button", { name: "Career" });
  }

  get careerLink() {
    return this.page.getByRole("link", { name: "Career" });
  }

  get talentProgrammeLink() {
    return this.page.getByRole("link", { name: "Talent programmes" });
  }

  async click() {
    await this.self.click();
  }

  async clickCareer() {
    await this.click();
    await this.careerLink.click();
  }

  async clickTalentProgramme() {
    await this.click();
    await this.talentProgrammeLink.click();
  }
}

// navigation-bar.ts
export class NavigationBar extends PageObject {
  constructor(
    page: Page,
    private readonly careerDropdownMenu: CareerDropdownMenu
  ) {
    super(page);
  }

  get contactsLink() {
    return this.page.getByRole("link", { name: "Contacts" });
  }

  get careerDropdownMenuButton() {
    return this.careerDropdownMenu.self;
  }

  async goToContacts() {
    await this.contactsLink.click();
  }

  async goToCareer() {
    await this.careerDropdownMenu.clickCareer();
  }

  async goToTalentProgramme() {
    await this.careerDropdownMenu.clickTalentProgramme();
  }
}
```

## Using Test Fixtures For Initializing Objects

Playwright Test's [fixture](https://playwright.dev/docs/test-fixtures) mechanism is a great way to reduce duplication of test setup for initializing objects and injecting dependencies.

```typescript
// ./fixtures/page-object-test.ts
interface PageObject {
  loginForm: LoginForm;
  careerDropdownMenu: CareerDropdownMenu;
  navigationBar: NavigationBar;
}

export const test = base.extend<PageObject>({
  loginForm: async ({ page }, use) => {
    await use(new LoginForm(page));
  },
  careerDropdownMenu: async ({ page }, use) => {
    await use(new CareerDropdownMenu(page));
  },
  navigationBar: async ({ page, careerDropdownMenu }, use) => {
    await use(new NavigationBar(page, careerDropdownMenu));
  },
});
```

## Defining Test Scenarios

When writing tests, describe the behavior of a specific feature or component of the application. The outermost describe block should provide the name of the feature or component being tested, and any nested describe blocks contribute to the test scenario’s name. Use clear and descriptive test titles that clearly state what is the expected outcome.

For example:

```javascript
/**
 * feature: Authentication
 * scenario: when a user has admin privileges it should redirect to the admin page
 * */
describe("Authentication", () => {
  describe("when a user has admin privileges", () => {
    it("should redirect to the admin page", async () => {
      // code
    });
  });
});
```
