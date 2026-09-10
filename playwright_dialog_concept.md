# How to Handle Dialog Box in Playwright

By default, **Playwright automatically dismisses native JavaScript dialogs** (like `alert`, `confirm`, and `prompt`), so your tests won't freeze if one appears unexpectedly. However, if you need to validate the message, accept an alert, type into a prompt, or click cancel, you must **register a custom dialog handler**. [[1](https://playwright.dev/docs/dialogs), [2](https://nareshit.com/blogs/handling-alerts-pop-ups-and-dialogs-in-playwright), [3](https://www.youtube.com/watch?v=V_NdrUUoz7U)]

## The Golden Rule: Register the Listener First

You must **always set up the dialog listener before** performing the action (like clicking a button) that triggers the pop-up. If you register it after, the event will be missed, and your action might stall or execute default behavior. [[1](https://www.youtube.com/watch?v=B9L0mq2TO_Q\&t=1056), [2](https://playwright.dev/docs/dialogs), [3](https://www.youtube.com/watch?v=V_NdrUUoz7U)]

## 1. Handling a Simple Alert

An `alert()` box simply displays a message and an "OK" button. You can use `dialog.accept()` to simulate clicking "OK". [[1](https://www.youtube.com/watch?v=_q25VaWFMzU\&t=30), [2](https://nareshit.com/blogs/handling-alerts-pop-ups-and-dialogs-in-playwright)]

```

// 1. Register the listener first
page.on('dialog', async dialog => {
console.log(dialog.message()); // Logs the alert text
await dialog.accept();         // Clicks the "OK" button
});

// 2. Trigger the action that opens the alert
await page.locator('#alertButton').click();

```

## 2. Handling Confirmation Dialogs

A `confirm()` box asks for user decision, providing both "OK" and "Cancel" buttons. [[1](https://nareshit.com/blogs/handling-alerts-pop-ups-and-dialogs-in-playwright)]

- Use `dialog.accept()` to click **OK**.
- Use `dialog.dismiss()` to click **Cancel**. [[1](https://www.youtube.com/watch?v=_q25VaWFMzU\&t=30)]

```

// Register listener to dismiss (Cancel) the confirmation
page.on('dialog', async dialog => {
if (dialog.type() === 'confirm') {
await dialog.dismiss(); // Clicks "Cancel"
}
});

await page.locator('#deleteRecordButton').click();

```

## 3. Handling Prompt Dialogs (With Input)

A `prompt()` box requests text input from the user. To pass a value into the text field, provide a string argument inside `dialog.accept()`. [[1](https://www.youtube.com/watch?v=_q25VaWFMzU\&t=30), [2](https://nareshit.com/blogs/handling-alerts-pop-ups-and-dialogs-in-playwright)]

```

page.on('dialog', async dialog => {
// Check the type to be safe
if (dialog.type() === 'prompt') {
await dialog.accept('John Doe'); // Types text and clicks "OK"
}
});

await page.locator('#promptButton').click();

```

## 4. One-Time Dialog Handling

If you only expect a dialog to appear exactly once and want to cleanly handle it without leaving a persistent listener running for the entire page lifecycle, use `page.once()` instead of `page.on()`. [[1](https://www.youtube.com/watch?v=_q25VaWFMzU\&t=30)]

```

// Fires only for the very next dialog event
page.once('dialog', async dialog => {
await dialog.accept();
});

await page.locator('#singleAlertButton').click();

```

## ⚠️ Crucial Distinction: Native Dialogs vs. HTML Modals

Be sure you are dealing with a native browser dialog.

- **Native Dialogs (****`alert`****,** **`confirm`****,** **`prompt`****):** These cannot be inspected via the browser's Developer Tools DOM tree. They require the `page.on('dialog')` event outlined above.
- **HTML Modals (Bootstrap, Tailwind, React Modals,** **`<dialog>`** **tags):** These *are* part of the page DOM. Do **not** use the dialog event listener for them. Instead, interact with them directly using standard Playwright locators (e.g., `await page.getByRole('button', { name: 'Close' }).click()`). [[1](https://playwright.dev/java/docs/dialogs), [2](https://www.youtube.com/watch?v=8pQmg9jckLo), [3](https://www.youtube.com/watch?v=v1JE3vgK6bM\&t=460), [4](https://www.youtube.com/watch?v=B9L0mq2TO_Q\&t=1056), [5](https://nareshit.com/blogs/handling-alerts-pop-ups-and-dialogs-in-playwright)]

If you want to refine this, let me know:

- What **programming language** are you using? (JavaScript/TypeScript, Python, C#, or Java?)
- Are you dealing with a **native browser pop-up** or a **custom HTML modal overlay**?

I can tailor a specific code block with exact test assertions for your context.

Playwright

**Dialogs | Playwright**

By default, dialogs are auto-dismissed by Playwright. However, you can register a dialog handler to: \* Accept the dialog \* Dismiss...

YouTube·Testers Talk

**Playwright TypeScript #52 Handling Alerts, Popup Windows in ...**

and I'll paste it in the new spec. file. so that's it guys so let me maximize this window. now. so I will delete the rest of the c...

Naresh IT

**Handling Alerts, Pop-ups & Dialogs in Playwright | NareshIT**

Table\_title: 2. Understanding the Different Types of Dialogs Table\_content: | Dialog Type | Example | Purpose | | --- | --- | --- ...

## You Said: Explain What Is `page.on()` Method / Function Here

explain what is page.on() method / function here

In Playwright, `page.on()` is **an event listener registration method**. It tells Playwright to actively "listen" for a specific event to happen on the browser page, and immediately execute a function (called a callback) whenever that event occurs.

It follows the standard **Node.js EventEmitter** pattern.

---

## How It Works (Line-by-Line Breakdown)

Consider this common dialog snippet:

```

// 1. Setup the listener
page.on('dialog', async dialog => {
console.log(dialog.message());
await dialog.accept();
});

// 2. Trigger the event
await page.locator('#button').click();

```

Here is exactly what is happening under the hood:

- **`page.on(...)`**: You are telling Playwright: *"Start watching this specific browser page."*
- **`'dialog'`**: This is the **Event Name**. You are specifying *what* you are watching for. In this case, you are telling Playwright to wake up whenever the browser fires a native JavaScript dialog (`alert`, `confirm`, or `prompt`).
- **`async dialog => { ... }`**: This is the **Callback Function**. It is the action plan.
- Playwright automatically creates a `dialog` object and passes it into this function.
- This object contains information about the pop-up (like `dialog.message()`) and methods to control it (like `dialog.accept()`).

## The Timeline: Why Order Matters

Because `page.on()` is **asynchronous** and event-driven, it acts like setting a trap.

1. **Registration:** When the code execution hits `page.on()`, it does **not** stop or pause the test. It simply registers the listener in the background and immediately moves to the next line of code.
2. **Waiting:** The test then executes the click action (`await page.locator(...).click()`).
3. **Trigger & Execution:** The moment the browser opens the dialog box, Playwright instantly catches the event, pauses what it was doing, runs the code inside your callback function (accepting the alert), and then resumes the test.

If you reverse the order and click the button *before* setting up `page.on()`, the dialog will appear, Playwright won't be listening for it, and your test will hang or fail.

## Difference Between `page.on()` and `page.once()`

- **`page.on()`** **(Persistent):** Keeps listening indefinitely. If your page triggers 5 alerts in a row, the function will run 5 separate times.
- **`page.once()`** **(One-time use):** Listens for the event exactly **once**. The moment the first dialog appears, it handles it and immediately destroys the listener. If a second alert pops up later, it will be ignored.

If you'd like, I can show you how to write this in **Python** or **Java** if you aren't using JavaScript, or explain how to **remove/turn off** the listener later in your test. Which would be most helpful?

## You Said: Playwright Instantly Catches the Event, Pauses What It Was Doing — Here What Pause? Anyways Post `click()`, We Are Not Doing Anything

Playwright instantly catches the event, pauses what it was doing - here what pause ? anyways post click(), we are not doing anything

You are completely right, and that is a very sharp catch! In that specific snippet, nothing comes after the `click()`, so there is technically nothing to pause.

However, the "pause" concept becomes critical when your test **does** have steps after the click. To understand exactly how Playwright handles this under the hood, we have to look at how asynchronous JavaScript promises execute.

## The Problem: Click Actions Wait for the Page to Settle

By default, when you run `await page.locator('#button').click()`, Playwright doesn't just click and instantly move on. It **waits for the click action to fully resolve** (which usually means waiting for network requests to finish, animations to stop, or the page to become idle).

When a native browser dialog (`alert`) pops up, the browser completely freezes the page execution. Because the page freezes, Playwright's `click()` command gets "stuck" in a pending state, waiting for the browser to become responsive again.

## What Playwright Actually Does Behind the Scenes

This is where the background event listener comes to the rescue. Here is the exact order of operations when you have steps *after* the click:

```

// 1. Background listener is set up
page.on('dialog', async dialog => {
await dialog.accept(); // <-- Captures and handles the dialog asynchronously
});

// 2. This starts, but gets "stuck" pending because the browser dialog freezes the page
await page.locator('#button').click();

// 3. Next steps in your test
await expect(page.locator('#success-message')).toBeVisible();

```

1. **`click()`** **starts:** Playwright clicks the button.
2. **The Dialog opens:** The browser freezes. The `await click()` line is now stuck waiting.
3. **The Event Interrupts:** In the background, browser events are still firing. Playwright detects the `'dialog'` event and instantly runs your callback (`dialog.accept()`).
4. **The Unblock:** The dialog closes, the browser unfreezes, and the `click()` command finally finishes resolving.
5. **The Test Resumes:** Only now does Playwright move on to line 10 (`await expect(...)`).

So, when we say it "pauses," it's not pausing your test code line-by-line; rather, the browser dialog blocks the `click()` action from finishing. The background `page.on()` handler is what unblocks the test so your subsequent lines can actually run!

Would you like to see how to use **`Promise.all()`** to cleanly bundle the click and the dialog handling together, or are you ready to look at **asserting the text** inside the dialog?
