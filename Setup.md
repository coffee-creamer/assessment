# Setup and Cleanup Strategy

## Purpose

The goal of setup and cleanup is to make the automated tests repeatable, isolated, and reliable. Each test should start from a known state and should not depend on data left behind by another test or another tester.

## Main Risk

The client currently uses shared users and a shared test environment. This creates a high risk of flaky automation because:

- A user account may already be logged in or locked.
- A cart may already contain items.
- Another tester may change the same data during execution.
- Orders may appear more than once.
- The wrong state may appear after refresh.
- Other teams may deploy or change the environment during test execution.

Automation should not simply reproduce this unstable manual process. It should control state as much as possible.

## Setup Approach

Before each test, the automation should create a clean and predictable starting point.

Recommended setup steps:

1. Start a fresh browser context for each test.
2. Navigate to the application login page.
3. Use a dedicated automation user where possible.
4. Log in with known credentials.
5. Reset application state if the application provides a reset option.
6. Confirm the cart is empty before continuing.
7. Use known product data and known checkout details.
8. Avoid relying on shared manual preconditions.

## Cleanup Approach

After each test, the automation should remove or isolate any state created during execution.

Recommended cleanup steps:

1. Close the browser context after each test.
2. Clear cookies, local storage, and session storage automatically by using a new context per test.
3. Remove cart items if the test leaves the cart populated.
4. Reset the application state where supported.
5. Avoid reusing a polluted user state across tests.
6. Record any data that could not be cleaned up as a known limitation.

## Playwright Setup and Cleanup Example

```ts
import { test, expect } from '@playwright/test';

test.describe('SauceDemo setup and cleanup example', () => {
  test.beforeEach(async ({ page }) => {
    // Start each test from the login page.
    // Playwright creates an isolated browser context per test by default.
    await page.goto('https://www.saucedemo.com/');
  });

  test.afterEach(async ({ page }) => {
    // Best-effort cleanup.
    // If the user is logged in, reset app state and log out.
    const menuButton = page.locator('#react-burger-menu-btn');

    if (await menuButton.isVisible().catch(() => false)) {
      await menuButton.click();

      const resetLink = page.locator('[data-test="reset-sidebar-link"]');
      if (await resetLink.isVisible().catch(() => false)) {
        await resetLink.click();
      }

      const logoutLink = page.locator('[data-test="logout-sidebar-link"]');
      if (await logoutLink.isVisible().catch(() => false)) {
        await logoutLink.click();
      }
    }
  });

  test('happy path purchase starts from a clean state', async ({ page }) => {
    await page.locator('[data-test="username"]').fill('standard_user');
    await page.locator('[data-test="password"]').fill('secret_sauce');
    await page.locator('[data-test="login-button"]').click();

    await expect(page).toHaveURL(/inventory/);
    await expect(page.locator('[data-test="title"]')).toContainText('Products');

    // Confirm the cart is clean before adding a product.
    await expect(page.locator('[data-test="shopping-cart-badge"]')).toHaveCount(0);

    await page.locator('[data-test="add-to-cart-sauce-labs-backpack"]').click();
    await expect(page.locator('[data-test="shopping-cart-badge"]')).toHaveText('1');
  });
});
```

## Why This Matters

Good setup and cleanup reduce false failures. If a test fails, the team should be confident that the failure is caused by a product issue or a real automation issue, not by leftover cart data, shared users, locked accounts, or another team changing the environment during execution.

## Recommended Controls

- Use dedicated automation accounts.
- Use a fresh browser context per test.
- Reset app state before or after each test.
- Avoid shared records where possible.
- Keep test data deterministic.
- Avoid fixed waits and rely on visible UI states and assertions.
- Track environment deployments so failures can be linked to release changes.

## Known Limitation

If the environment is shared and other teams can change data during execution, cleanup alone will not fully remove flakiness. The longer-term solution is either environment isolation, dedicated automation data, or a controlled execution window.
