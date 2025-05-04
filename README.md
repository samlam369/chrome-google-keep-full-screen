# Overview

This repo is forked from [chrisputnam9/chrome-google-keep-full-screen](https://github.com/chrisputnam9/chrome-google-keep-full-screen). This document details the modifications made to the Google Keep Full Screen extension to make it compatible with Electron. Since this is now a fork of the original extension hosted at [samlam369/chrome-google-keep-full-screen](https://github.com/samlam369/chrome-google-keep-full-screen), this information will help future maintainers understand the changes and maintain them when merging upstream updates.

## Why Modifications Are Needed

Electron provides Chrome extension API compatibility, but there are some differences in how these APIs behave compared to a regular Chrome browser:

1. **Availability of Chrome APIs**: Some Chrome APIs may not be fully available or might behave differently in Electron.
2. **Error Handling**: The extension needs robust error handling to gracefully handle cases where Chrome APIs are absent or fail.
3. **DOM Element Availability**: The extension needs to check for element existence before attaching observers or event listeners.

## Key Modifications

### 1. Main Initialization Function

The `init()` function has been wrapped in a try/catch block to prevent initialization failures from crashing the entire extension:

```javascript
init: async function () {
    try {
        // Original initialization code
        ...
    } catch (err) {
        console.error('Extension initialization error:', err);
    }
},
```

### 2. Chrome Storage API Handling

The Chrome storage API calls are wrapped in try/catch blocks with fallbacks:

```javascript
// Original code
const storage = await promise_chrome_storage_sync_get(["settings"]);

// Modified code
try {
    const storage = await promise_chrome_storage_sync_get(["settings"]);
    if ("settings" in storage && "fullscreen" in storage.settings) {
        this.fullscreen = storage.settings.fullscreen;
    }
} catch (storageErr) {
    // Chrome storage may not be available in Electron
    console.log('Chrome storage API error:', storageErr);
    // Continue with default fullscreen setting
}
```

### 3. Chrome Messaging API Handling

Chrome messaging is wrapped in try/catch to handle potential unavailability:

```javascript
// Original code
chrome.runtime.onMessage.addListener(function (request) {
    // Handle keyboard shortcuts
    ...
});

// Modified code
try {
    chrome.runtime.onMessage.addListener(function (request) {
        // Handle keyboard shortcuts
        ...
    });
} catch (chromeErr) {
    // Chrome messaging may not be available in Electron
    console.log('Chrome messaging API error:', chromeErr);
}
```

### 4. DOM Element Existence Checks

Added checks for element existence before attaching observers:

```javascript
// Original code
main.observerNewNotes.observe(elCreatedNotesGroupContainer, {
    childList: true,
});

// Modified code
if (elCreatedNotesGroupContainer) {
    main.observerNewNotes.observe(elCreatedNotesGroupContainer, {
        childList: true,
    });
} else {
    console.log('Note container not found, observer not attached');
}
```

### 5. Promisified Chrome API Methods

The `promise_chrome_storage_sync_set` and `promise_chrome_storage_sync_get` methods have been modified to handle errors gracefully:

```javascript
// Original
function promise_chrome_storage_sync_get(data) {
    return new Promise((resolve, reject) => {
        try {
            chrome.storage.sync.get(data, resolve);
        } catch (error) {
            reject(error);
        }
    });
}

// Modified
function promise_chrome_storage_sync_get(data) {
    return new Promise((resolve, reject) => {
        try {
            chrome.storage.sync.get(data, resolve);
        } catch (error) {
            console.log("chrome.storage.sync.get error:", error);
            // Return empty object to prevent errors in Electron
            resolve({});
        }
    });
}
```

### 6. Options Menu Event Handler

The options menu click handler is also wrapped in a try/catch:

```javascript
// Original
elMenuItemOptions.addEventListener("click", function (event) {
    event.preventDefault();
    chrome.runtime.sendMessage({ action: "open-options" });
});

// Modified
elMenuItemOptions.addEventListener("click", function (event) {
    event.preventDefault();
    try {
        chrome.runtime.sendMessage({ action: "open-options" });
    } catch (error) {
        console.log("Chrome runtime sendMessage error:", error);
    }
});
```

## Maintenance Strategy

As this is now a fork maintained at [samlam369/chrome-google-keep-full-screen](https://github.com/samlam369/chrome-google-keep-full-screen), changes from the upstream repository should be manually merged into this fork. When merging upstream changes:

1. Pull the upstream changes into a separate branch
2. Review the changes, particularly any modifications to `script.js`
3. Manually merge the changes while preserving the error handling modifications
4. Run the app to test that the extension still works correctly in Electron

## Manifest Management

The Electron app expects the extension's `manifest.json` to be present and properly configured in the root of this directory. For compatibility and to ensure the extension loads correctly:

- The `manifest.json` is copied from `publish/manifest-chrome.json` (or created from the latest upstream manifest if needed).
- The `run_at` property for the content script must be set to `"document_idle"`.
- If you update the manifest or pull changes from upstream, always verify that `run_at` is still set correctly in `manifest.json`.
- This ensures that the extension is injected at the appropriate time for Google Keep to function correctly in Electron.

If you make changes to the manifest or update the extension, make sure to check this property before running or distributing the app.

## Testing Changes

After making any changes to the extension code, test the following functionality:

1. Opening the app and ensuring the extension loads
2. Creating and opening a note to verify the full-screen functionality works
3. Toggling full-screen mode using the button in the note toolbar
4. Testing any keyboard shortcuts defined by the extension

By maintaining our own fork with these compatibility changes, we ensure the extension will continue to work reliably in our Electron application while still allowing us to incorporate improvements from the upstream repository.