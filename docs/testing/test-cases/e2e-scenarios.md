# Test Cases: End-to-End Scenarios

**Version:** 1.0
**Last Updated:** 2026-02-02
**Test Framework:** Playwright

---

## Overview

End-to-end tests verify complete user journeys across all services. These tests use real browsers and interact with a fully deployed staging environment.

**Principles:**
- Test critical user journeys only (~50 tests)
- Use realistic test data
- Independent, parallelizable tests
- Self-cleaning (no manual data cleanup)

---

## Test Categories

1. [Authentication](#1-authentication)
2. [Messaging](#2-messaging)
3. [Real-Time Features](#3-real-time-features)
4. [Channels](#4-channels)
5. [Search](#5-search)
6. [File Uploads](#6-file-uploads)
7. [Multi-Device](#7-multi-device)
8. [Notifications](#8-notifications)

---

## 1. Authentication

### TC-E2E-AUTH-001: User Login with Keycloak

**Priority:** Critical
**Duration:** ~10s

**Steps:**
1. Navigate to login page
2. Enter valid credentials
3. Click login button
4. Verify redirect to channel list

**Playwright Test:**
```typescript
test('user can login with valid credentials', async ({ page }) => {
  await page.goto('/login');

  await page.fill('[data-testid="email"]', 'alice@test.com');
  await page.fill('[data-testid="password"]', 'TestPassword123!');
  await page.click('[data-testid="login-button"]');

  await expect(page).toHaveURL('/channels');
  await expect(page.locator('[data-testid="user-menu"]')).toContainText('Alice');
});
```

**Expected Results:**
- User redirected to /channels
- User menu shows username
- JWT stored in localStorage/cookie

---

### TC-E2E-AUTH-002: User Login with Invalid Credentials

**Priority:** High
**Duration:** ~5s

**Steps:**
1. Navigate to login page
2. Enter invalid credentials
3. Click login button

**Expected Results:**
- Error message displayed
- User remains on login page
- No JWT stored

---

### TC-E2E-AUTH-003: User Logout

**Priority:** High
**Duration:** ~5s

**Steps:**
1. Login as user
2. Click user menu
3. Click logout

**Expected Results:**
- Redirected to login page
- JWT cleared
- WebSocket disconnected

---

### TC-E2E-AUTH-004: Session Expiry

**Priority:** Medium
**Duration:** ~15s (with wait)

**Steps:**
1. Login as user
2. Wait for session to expire (or manually expire)
3. Attempt to load messages

**Expected Results:**
- Redirected to login page
- Appropriate message shown

---

## 2. Messaging

### TC-E2E-MSG-001: Send and Receive Message

**Priority:** Critical
**Duration:** ~15s

**Steps:**
1. Alice logs in on Browser 1
2. Bob logs in on Browser 2
3. Both open #general channel
4. Alice sends message
5. Verify Bob receives message in real-time

**Playwright Test:**
```typescript
test('user can send message and other user receives it', async ({ browser }) => {
  // Setup two browser contexts
  const aliceContext = await browser.newContext();
  const bobContext = await browser.newContext();
  const alicePage = await aliceContext.newPage();
  const bobPage = await bobContext.newPage();

  // Login both users
  await loginUser(alicePage, 'alice@test.com', 'password');
  await loginUser(bobPage, 'bob@test.com', 'password');

  // Both open #general
  await alicePage.click('[data-testid="channel-general"]');
  await bobPage.click('[data-testid="channel-general"]');

  // Alice sends message
  const testMessage = `E2E test ${Date.now()}`;
  await alicePage.fill('[data-testid="message-input"]', testMessage);
  await alicePage.press('[data-testid="message-input"]', 'Enter');

  // Verify Alice sees her message
  await expect(alicePage.locator(`text=${testMessage}`)).toBeVisible();

  // Verify Bob receives in real-time (< 5s)
  await expect(bobPage.locator(`text=${testMessage}`)).toBeVisible({ timeout: 5000 });

  // Cleanup
  await aliceContext.close();
  await bobContext.close();
});
```

**Expected Results:**
- Message appears in Alice's view immediately
- Message appears in Bob's view within 5 seconds
- Message persisted (visible after refresh)

---

### TC-E2E-MSG-002: Send Message with Mention

**Priority:** High
**Duration:** ~15s

**Steps:**
1. Alice sends message "@bob check this"
2. Verify Bob sees notification indicator
3. Verify mention is highlighted

**Expected Results:**
- Bob's channel shows unread mention indicator
- @bob is clickable and highlighted
- Bob receives push notification (if enabled)

---

### TC-E2E-MSG-003: Edit Message

**Priority:** High
**Duration:** ~10s

**Steps:**
1. Alice sends message
2. Alice hovers message, clicks edit
3. Alice modifies text and saves
4. Verify Bob sees edited message

**Expected Results:**
- Message updated in both views
- "Edited" indicator shown
- Edit history accessible

---

### TC-E2E-MSG-004: Delete Message

**Priority:** High
**Duration:** ~10s

**Steps:**
1. Alice sends message
2. Alice hovers message, clicks delete
3. Confirms deletion
4. Verify Bob sees message removed

**Expected Results:**
- Message removed from both views
- "Message deleted" placeholder shown (or removed entirely)

---

### TC-E2E-MSG-005: Reply in Thread

**Priority:** High
**Duration:** ~15s

**Steps:**
1. Alice sends message in #general
2. Bob clicks "Reply in thread"
3. Bob sends thread reply
4. Alice clicks thread indicator

**Expected Results:**
- Thread panel opens
- Reply visible in thread
- Thread count updated on parent message

---

### TC-E2E-MSG-006: Message with Code Block

**Priority:** Medium
**Duration:** ~10s

**Steps:**
1. Alice sends message with ```code block```
2. Verify syntax highlighting applied

**Expected Results:**
- Code block rendered with formatting
- Language-specific highlighting (if specified)
- Copy button available

---

### TC-E2E-MSG-007: Message with Link Preview

**Priority:** Medium
**Duration:** ~20s

**Steps:**
1. Alice sends message with URL
2. Wait for link preview to load

**Expected Results:**
- Link preview appears below message
- Shows title, description, image (if available)
- Preview loads asynchronously

---

### TC-E2E-MSG-008: Load Message History

**Priority:** High
**Duration:** ~10s

**Steps:**
1. Open channel with 1000+ messages
2. Scroll to top
3. Verify older messages load

**Expected Results:**
- Initial load shows recent messages
- Scrolling up loads older messages
- Loading indicator shown during fetch
- No duplicate messages

---

## 3. Real-Time Features

### TC-E2E-RT-001: Typing Indicator

**Priority:** High
**Duration:** ~15s

**Steps:**
1. Alice and Bob in same channel
2. Alice starts typing
3. Verify Bob sees "Alice is typing..."
4. Alice stops typing
5. Verify indicator disappears

**Playwright Test:**
```typescript
test('typing indicator appears and disappears', async ({ browser }) => {
  const [alicePage, bobPage] = await setupTwoUsers(browser);

  // Both in #general
  await alicePage.click('[data-testid="channel-general"]');
  await bobPage.click('[data-testid="channel-general"]');

  // Alice starts typing
  await alicePage.fill('[data-testid="message-input"]', 'Hello');

  // Bob sees typing indicator
  await expect(bobPage.locator('[data-testid="typing-indicator"]'))
    .toContainText('Alice is typing');

  // Alice clears input
  await alicePage.fill('[data-testid="message-input"]', '');

  // Indicator disappears (with timeout for debounce)
  await expect(bobPage.locator('[data-testid="typing-indicator"]'))
    .not.toBeVisible({ timeout: 6000 });
});
```

**Expected Results:**
- Indicator appears within 1s of typing
- Indicator disappears within 5s of stopping
- Multiple typers shown: "Alice and Bob are typing..."

---

### TC-E2E-RT-002: User Presence

**Priority:** High
**Duration:** ~20s

**Steps:**
1. Alice logs in
2. Bob views user list
3. Verify Alice shows as online
4. Alice logs out
5. Verify Alice shows as offline

**Expected Results:**
- Online status updates within 5s
- Offline status updates within 30s
- Status indicator (green dot, etc.)

---

### TC-E2E-RT-003: Read Receipts (DM)

**Priority:** High
**Duration:** ~15s

**Steps:**
1. Alice sends DM to Bob
2. Bob opens the DM
3. Verify Alice sees "Read" indicator

**Expected Results:**
- "Delivered" shown when Bob receives
- "Read" shown when Bob opens DM
- Timestamp shown on hover

---

### TC-E2E-RT-004: Unread Count Updates

**Priority:** High
**Duration:** ~15s

**Steps:**
1. Bob in #general, Alice in #random
2. Bob sends message to #random
3. Verify Alice sees unread badge
4. Alice opens #random
5. Verify badge clears

**Expected Results:**
- Unread count increments in real-time
- Badge clears when channel opened
- Total unread shown in app icon/title

---

## 4. Channels

### TC-E2E-CH-001: Create Public Channel

**Priority:** High
**Duration:** ~15s

**Steps:**
1. Click "Create Channel"
2. Enter name "test-channel"
3. Select "Public"
4. Click Create
5. Verify channel appears in list

**Expected Results:**
- Channel created and selected
- User is channel owner
- Channel visible in public list

---

### TC-E2E-CH-002: Create Private Channel

**Priority:** High
**Duration:** ~15s

**Steps:**
1. Create private channel
2. Add specific members
3. Verify non-members can't see channel

**Expected Results:**
- Channel not in public list
- Only invited members can access
- Invite link works

---

### TC-E2E-CH-003: Join Public Channel

**Priority:** High
**Duration:** ~10s

**Steps:**
1. Browse public channels
2. Click "Join" on a channel
3. Verify membership

**Expected Results:**
- User added to channel
- Channel appears in user's list
- Can send messages

---

### TC-E2E-CH-004: Leave Channel

**Priority:** Medium
**Duration:** ~10s

**Steps:**
1. Open channel settings
2. Click "Leave Channel"
3. Confirm

**Expected Results:**
- Channel removed from list
- Can't send messages
- Can rejoin (if public)

---

### TC-E2E-CH-005: Channel Settings Update

**Priority:** Medium
**Duration:** ~10s

**Steps:**
1. Open channel settings (as owner)
2. Change name and description
3. Save

**Expected Results:**
- Changes reflected immediately
- All members see update
- Audit log entry created

---

## 5. Search

### TC-E2E-SRCH-001: Search Messages

**Priority:** High
**Duration:** ~15s

**Steps:**
1. Open search
2. Enter search query
3. Verify results

**Playwright Test:**
```typescript
test('user can search for messages', async ({ page }) => {
  await loginUser(page, 'alice@test.com', 'password');

  // Send a unique message first
  const uniqueText = `searchable-${Date.now()}`;
  await page.click('[data-testid="channel-general"]');
  await page.fill('[data-testid="message-input"]', uniqueText);
  await page.press('[data-testid="message-input"]', 'Enter');

  // Wait for indexing
  await page.waitForTimeout(2000);

  // Search for the message
  await page.click('[data-testid="search-button"]');
  await page.fill('[data-testid="search-input"]', uniqueText);
  await page.press('[data-testid="search-input"]', 'Enter');

  // Verify result
  await expect(page.locator('[data-testid="search-results"]'))
    .toContainText(uniqueText);
});
```

**Expected Results:**
- Results displayed with context
- Clicking result jumps to message
- Highlighting shows matched terms

---

### TC-E2E-SRCH-002: Search with Filters

**Priority:** Medium
**Duration:** ~15s

**Steps:**
1. Search with channel filter
2. Search with date range
3. Search with sender filter

**Expected Results:**
- Filters correctly applied
- Results match criteria

---

## 6. File Uploads

### TC-E2E-FILE-001: Upload Image

**Priority:** High
**Duration:** ~20s

**Steps:**
1. Click attachment button
2. Select image file
3. Wait for upload
4. Send message

**Expected Results:**
- Upload progress shown
- Thumbnail generated
- Image viewable in lightbox

---

### TC-E2E-FILE-002: Upload Document

**Priority:** Medium
**Duration:** ~20s

**Steps:**
1. Upload PDF document
2. Verify preview

**Expected Results:**
- Document attached
- Preview available (if supported)
- Download link works

---

### TC-E2E-FILE-003: Drag and Drop Upload

**Priority:** Medium
**Duration:** ~15s

**Steps:**
1. Drag file to message area
2. Drop file

**Expected Results:**
- Drop zone highlighted
- File uploads
- Can add caption before sending

---

## 7. Multi-Device

### TC-E2E-MULTI-001: Read Sync Across Devices

**Priority:** High
**Duration:** ~20s

**Steps:**
1. Alice logged in on Desktop and Mobile (2 browser contexts)
2. Bob sends message to Alice
3. Alice reads on Desktop
4. Verify Mobile shows as read

**Playwright Test:**
```typescript
test('read state syncs across devices', async ({ browser }) => {
  // Alice on two "devices"
  const desktopContext = await browser.newContext({ viewport: { width: 1920, height: 1080 } });
  const mobileContext = await browser.newContext({ viewport: { width: 375, height: 667 } });

  const desktopPage = await desktopContext.newPage();
  const mobilePage = await mobileContext.newPage();

  await loginUser(desktopPage, 'alice@test.com', 'password');
  await loginUser(mobilePage, 'alice@test.com', 'password');

  // Both show unread in #general
  await expect(desktopPage.locator('[data-testid="channel-general"] .unread-badge'))
    .toBeVisible();
  await expect(mobilePage.locator('[data-testid="channel-general"] .unread-badge'))
    .toBeVisible();

  // Read on desktop
  await desktopPage.click('[data-testid="channel-general"]');

  // Verify mobile badge clears
  await expect(mobilePage.locator('[data-testid="channel-general"] .unread-badge'))
    .not.toBeVisible({ timeout: 5000 });
});
```

**Expected Results:**
- Unread badge clears on both devices
- Read pointer synced
- Within 5 seconds

---

### TC-E2E-MULTI-002: Draft Sync Across Devices

**Priority:** Medium
**Duration:** ~15s

**Steps:**
1. Start typing on Desktop
2. Switch to Mobile
3. Verify draft appears

**Expected Results:**
- Draft synced within 5 seconds
- Can continue typing on other device

---

## 8. Notifications

### TC-E2E-NOTIF-001: Browser Notification on Mention

**Priority:** Medium
**Duration:** ~15s

**Preconditions:**
- Browser notifications enabled

**Steps:**
1. Alice in background tab
2. Bob mentions Alice
3. Verify browser notification appears

**Expected Results:**
- Notification shown
- Clicking notification focuses tab
- Notification content shows sender and preview

---

### TC-E2E-NOTIF-002: No Notification When Active

**Priority:** Medium
**Duration:** ~15s

**Steps:**
1. Alice actively viewing channel
2. Bob sends message
3. Verify no notification (already visible)

**Expected Results:**
- No browser notification
- No sound (if configured)

---

## Test Execution Configuration

### Playwright Config

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  timeout: 60000,
  retries: 2,
  workers: 4,

  use: {
    baseURL: process.env.STAGING_URL || 'https://staging.chat.example.com',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    trace: 'retain-on-failure',
  },

  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    { name: 'mobile-chrome', use: { ...devices['Pixel 5'] } },
    { name: 'mobile-safari', use: { ...devices['iPhone 12'] } },
  ],

  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['junit', { outputFile: 'test-results/junit.xml' }],
  ],
});
```

### Test Data Helpers

```typescript
// tests/e2e/helpers.ts
export async function loginUser(page: Page, email: string, password: string) {
  await page.goto('/login');
  await page.fill('[data-testid="email"]', email);
  await page.fill('[data-testid="password"]', password);
  await page.click('[data-testid="login-button"]');
  await page.waitForURL('/channels');
}

export async function setupTwoUsers(browser: Browser): Promise<[Page, Page]> {
  const context1 = await browser.newContext();
  const context2 = await browser.newContext();
  const page1 = await context1.newPage();
  const page2 = await context2.newPage();

  await loginUser(page1, 'alice@test.com', 'password');
  await loginUser(page2, 'bob@test.com', 'password');

  return [page1, page2];
}

export function generateUniqueMessage(): string {
  return `e2e-test-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}
```

---

## Test Execution Summary

| Category | Total | Critical | High | Medium | Low |
|----------|-------|----------|------|--------|-----|
| Authentication | 4 | 1 | 2 | 1 | 0 |
| Messaging | 8 | 1 | 5 | 2 | 0 |
| Real-Time Features | 4 | 0 | 4 | 0 | 0 |
| Channels | 5 | 0 | 3 | 2 | 0 |
| Search | 2 | 0 | 1 | 1 | 0 |
| File Uploads | 3 | 0 | 1 | 2 | 0 |
| Multi-Device | 2 | 0 | 1 | 1 | 0 |
| Notifications | 2 | 0 | 0 | 2 | 0 |
| **Total** | **30** | **2** | **17** | **11** | **0** |

---

## CI Integration

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main]
  pull_request:

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Run E2E tests
        run: npx playwright test
        env:
          STAGING_URL: ${{ secrets.STAGING_URL }}

      - name: Upload report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
```
