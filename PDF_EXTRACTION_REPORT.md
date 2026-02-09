# Detailed Report: PDF Extraction Process in Zotero Connectors

## Executive Summary

This report provides a comprehensive analysis of how the Zotero Connectors browser extension extracts PDFs when activated on a journal article's webpage. The system employs a sophisticated multi-layered architecture involving web request interception, background processing, bot protection bypass mechanisms, and communication between content scripts and the Zotero client.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [PDF Detection Mechanisms](#pdf-detection-mechanisms)
3. [User Interaction Flow](#user-interaction-flow)
4. [PDF Download and Extraction](#pdf-download-and-extraction)
5. [Bot Protection Bypass System](#bot-protection-bypass-system)
6. [Data Transfer to Zotero](#data-transfer-to-zotero)
7. [Progress Tracking and UI Updates](#progress-tracking-and-ui-updates)
8. [Key Components and Files](#key-components-and-files)
9. [Complete Flow Diagram](#complete-flow-diagram)
10. [Technical Details](#technical-details)

---

## Architecture Overview

The Zotero Connector operates with three main components:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Journal Article Webpage                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │         Injected Content Scripts (inject.jsx)            │   │
│  │  - Translation framework                                 │   │
│  │  - Page saving orchestration (pageSaving.js)            │   │
│  └────────────────────────┬─────────────────────────────────┘   │
└───────────────────────────┼─────────────────────────────────────┘
                            │ Message Passing
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│              Background Process (background.js)                  │
│  - Web request interception (webRequestIntercept.js)            │
│  - PDF detection and UI updates                                 │
│  - Attachment download orchestration (itemSaver_background.js)  │
│  - Bot protection bypass (browserAttachmentMonitor.js)          │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTP/API Calls
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│              Zotero Standalone Client / zotero.org               │
│  - Connector HTTP Server (port 23119)                           │
│  - Item and attachment storage                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## PDF Detection Mechanisms

### 1. Web Request Interception (Primary Method)

**File:** `src/browserExt/webRequestIntercept.js`

The connector uses the browser's `webRequest` API to monitor all HTTP responses:

```javascript
// Initialization
browser.webRequest.onHeadersReceived.addListener(
    Zotero.WebRequestIntercept.handleRequest('headersReceived'),
    {urls: ['<all_urls>'], types: ["main_frame", "sub_frame"]},
    ["responseHeaders"]
);
```

**Detection Logic:**

```javascript
offerSavingPDFInFrame: function(details) {
    if (details.frameId === 0) return;
    if (!details.responseHeadersObject['content-type']) return;
    const contentType = details.responseHeadersObject['content-type'].split(';')[0];
    
    // If no translators are found for the top frame or the first child frame, 
    // and some frame contains a pdf, saving that PDF will be offered.
    if (contentType == 'application/pdf') {
        setTimeout(() => Zotero.Connector_Browser.onPDFFrame(
            details.url, 
            details.frameId, 
            details.tabId
        ));
    }
}
```

**Key Points:**
- Monitors `Content-Type` header in HTTP responses
- Detects `application/pdf` and `application/epub+zip` MIME types
- Only activates for sub-frames (frameId !== 0)
- Calls `onPDFFrame()` to initiate PDF save functionality

### 2. Content Script Detection (Safari Fallback)

**File:** `src/common/inject/pageSaving.js`

On Safari, where web request APIs are limited, detection occurs in the content script:

```javascript
async onPageLoad(force) {
    let translate = await this._initTranslate();
    let translators = await Zotero.TranslateWeb.detect({ translate });
    
    // Safari fallback: check document content type
    if (!translators.length && Zotero.isSafari) {
        if (!isTopWindow && document.contentType == 'application/pdf') {
            return Zotero.Connector_Browser.onPDFFrame(
                document.location.href, 
                instanceID
            );
        }
    }
}
```

---

## User Interaction Flow

### 1. Button Activation

**File:** `src/browserExt/background.js`

When a PDF is detected, the extension updates its UI:

```javascript
onPDFFrame(url, frameId, tabId) → 
    onTranslators(translators, instanceID, contentType) →
        _updateExtensionUI(tab)
```

**UI Changes:**
- Extension icon changes to PDF icon
- Toolbar button becomes clickable
- Context menu shows "Save to Zotero (PDF)" option
- Button tooltip updates to indicate PDF save capability

### 2. Save Initiation

When the user clicks the save button:

```javascript
// background.js
_browserAction: function(tab) {
    let tabInfo = this._tabInfo[tab.id];
    if (tabInfo.isPDF) {
        // Route to PDF save path
        Zotero.Messaging.sendMessage("saveAsWebpage", {
            snapshot: true
        }, tab);
    }
}
```

### 3. Message Routing to Content Script

**File:** `src/common/inject/pageSaving.js`

```javascript
onSaveAsWebpage: async function({ title, saveSnapshot } = {}) {
    // Check if document is PDF
    const isOnline = await Zotero.Connector.checkIsOnline();
    const supportsAttachmentUpload = await Zotero.Connector.getPref(
        'supportsAttachmentUpload'
    );

    // If content type is not text (e.g., application/pdf)
    if (!document.contentType.startsWith('text') && 
        (supportsAttachmentUpload || !isOnline)) {
        return await this._saveAsStandaloneAttachment({title, saveSnapshot});
    }
    return await this._saveAsWebpage({title, saveSnapshot});
}
```

---

## PDF Download and Extraction

### 1. Standalone Attachment Creation

**File:** `src/common/inject/pageSaving.js`

```javascript
async _saveAsStandaloneAttachment({ title=document.title } = {}) {
    const sessionID = this.sessionDetails.id;
    
    // Determine item type from content
    let itemType = "webpage";
    if (document.contentType === 'application/pdf') {
        itemType = "pdf"
    } else if (document.contentType === 'application/epub+zip') {
        itemType = "epub";
    }

    // Create attachment metadata
    let standaloneAttachment = {
        url: document.location.toString(),
        mimeType: document.contentType,
        title,
        linkMode: "imported_url",
        referrer: document.referrer
    }

    // Initiate download
    await Zotero.ItemSaver.saveStandaloneAttachmentToZotero(
        standaloneAttachment, 
        sessionID
    )
}
```

### 2. Attachment Fetching

**File:** `src/common/itemSaver_background.js`

The background script handles the actual HTTP request:

```javascript
Zotero.ItemSaver._fetchAttachment = async function(attachment, tab, attemptBotProtectionBypass=true) {
    let options = { 
        responseType: "arraybuffer", 
        timeout: 60000 
    };
    
    // Step 1: Retrieve cookies for authentication
    if (!Zotero.isSafari) {
        let cookies = await browser.cookies.getAll({
            url: attachment.url,
            partitionKey: {},
        });
        
        // Filter for partitioned cookies (e.g., Cloudflare clearance)
        cookies = cookies.filter(c => c.partitionKey);
        options.headers = {
            "Cookie": cookies.map(
                cookie => `${cookie.name}=${cookie.value}`
            ).join('; ')
        }
        options.referrer = attachment.referrer;
    }

    // Step 2: Attempt HTTP request
    try {
        let xhr = await Zotero.HTTP.request("GET", attachment.url, options);
        let result = Zotero.Utilities.Connector.getContentTypeFromXHR(xhr);
        receivedMimeType = result.contentType;

        // Step 3: Validate content type
        if (!attachment.mimeType) {
            // No expected type specified - accept whatever we received
            attachment.mimeType = receivedMimeType;
            return xhr.response;
        }
        
        // Verify MIME type matches expectation
        if (attachment.mimeType.toLowerCase() === receivedMimeType.toLowerCase()) {
            return xhr.response;
        }
        
        // Accept application/octet-stream for PDFs (some hosts)
        const expectedPDFIsOctetStream = 
            attachment.mimeType.toLowerCase() === 'application/pdf'
            && receivedMimeType.toLowerCase() === 'application/octet-stream'
        
        if (expectedPDFIsOctetStream) {
            return xhr.response;
        }
    } catch (e) {
        // Step 4: Handle download failures
        if (e.status === 404 || 
            !attemptBotProtectionBypass || 
            !tab || 
            !this._isUrlBotBypassWhitelisted(attachment.url)) {
            throw e;
        }
        
        Zotero.debug(`Error downloading attachment ${attachment.url}: ${e.message}`);
        Zotero.debug("Attempting bot protection bypass");
    }

    // Step 5: Attempt bot protection bypass (see next section)
    // ...
}
```

**Key Features:**
- Retrieves browser cookies including partitioned cookies (Cloudflare)
- Sets referrer header to bypass referrer checks
- Validates content type against expected MIME type
- Returns PDF data as ArrayBuffer
- Timeout after 60 seconds

---

## Bot Protection Bypass System

Many journal sites implement JavaScript-based bot detection, CAPTCHAs, or other protection mechanisms. Zotero implements two fallback strategies:

**Whitelisted Domains:**
```javascript
let BOT_BYPASS_WHITELISTED_DOMAINS = [
    'sciencedirect.com',
    'pdf.sciencedirectassets.com',
    'ncbi.nlm.nih.gov', // PubMed
];
```

### Strategy 1: Hidden Iframe Method

**File:** `src/common/itemSaver_background.js`

```javascript
Zotero.ItemSaver._passJSBotDetectionViaHiddenIframe = async function(url, tab) {
    const iframeUrl = Zotero.getExtensionURL(
        "browserAttachmentMonitor/browserAttachmentMonitor.html"
    );

    // Step 1: Inject hidden iframe into current page
    await browser.scripting.executeScript({
        target: { tabId: tab.id },
        func: (url, id) => {
            let iframe = document.createElement('iframe');
            iframe.style.display = 'none';
            iframe.src = url;
            iframe.id = id;
            document.body.appendChild(iframe);
        },
        args: [iframeUrl, id]
    });

    // Step 2: Monitor for attachment download
    let pdfURL = await Zotero.BrowserAttachmentMonitor.waitForAttachment(
        tab.id, 
        url, 
        5000 // 5 second timeout
    );
    
    // Step 3: Clean up iframe
    await browser.scripting.executeScript({
        target: { tabId: tab.id },
        func: (id) => {
            const iframe = document.getElementById(id);
            if (iframe) iframe.parentNode.removeChild(iframe);
        },
        args: [id]
    });
    
    return pdfURL;
}
```

**How it works:**
1. Creates an invisible iframe pointing to `browserAttachmentMonitor.html`
2. The iframe navigates to the PDF URL
3. JavaScript on the page executes, passing bot detection
4. Browser Attachment Monitor intercepts the successful PDF download
5. Returns the final PDF URL to the main download process

### Strategy 2: Popup Window Method

**File:** `src/common/itemSaver_background.js`

```javascript
Zotero.ItemSaver._passJSBotDetectionViaWindowPrompt = async function(url, tab) {
    // Step 1: Calculate centered window position
    const screenInfo = await browser.scripting.executeScript({
        target: { tabId: tab.id },
        func: () => ({
            width: window.screen.availWidth,
            height: window.screen.availHeight,
            left: window.screen.availLeft,
            top: window.screen.availTop
        })
    });
    
    const screen = screenInfo[0].result;
    const width = Math.floor(screen.width * 0.8);
    const height = Math.floor(screen.height * 0.8);
    const left = screen.left + Math.floor((screen.width - width) / 2);
    const top = screen.top + Math.floor((screen.height - height) / 2);
    
    // Step 2: Create popup window
    const monitorUrl = Zotero.getExtensionURL(
        "browserAttachmentMonitor/browserAttachmentMonitor.html"
    );
    const window = await browser.windows.create({
        url: monitorUrl,
        type: 'popup',
        width,
        height,
        left,
        top
    });
    
    // Step 3: Wait for user to solve CAPTCHA and PDF to download
    const pdfURL = await Zotero.BrowserAttachmentMonitor.waitForAttachment(
        window.tabs[0].id, 
        url
    );
    
    return pdfURL;
}
```

**How it works:**
1. Opens a popup window with the attachment monitor page
2. User can interact with CAPTCHA or bot detection challenges
3. Monitor intercepts successful PDF download
4. Returns final PDF URL

### Browser Attachment Monitor

**File:** `src/browserExt/browserAttachmentMonitor/browserAttachmentMonitor.js`

This component uses Declarative Net Request (Chrome) or webRequest API (Firefox) to intercept HTTP responses:

```javascript
Zotero.BrowserAttachmentMonitor = {
    waitForAttachment: async function(tabId, url, timeoutMs=60000) {
        const ruleId = this._nextRuleId++;

        // Setup message listener for success notification
        const messageListener = (message, sender) => {
            if (sender.tab.id === tabId && 
                message.type === 'attachment-monitor-loaded') {
                if (message.success) {
                    resolveSucceeded(message.success);
                    this._cleanup(ruleId, messageListener, tabRemovedListener);
                } else {
                    // Redirect to actual PDF URL
                    browser.tabs.sendMessage(tabId, { 
                        type: 'redirect-attachment-monitor', 
                        url 
                    });
                }
            }
        };

        // Add DNR rule to intercept PDF downloads
        if (Zotero.isFirefox) {
            await this._addWebRequestRule(tabId, ruleId);
        } else {
            await this._addDNRRule(tabId, ruleId);
        }
        
        return await Promise.race([
            successPromise,
            Zotero.Promise.delay(timeoutMs).then(() => { 
                throw new Error('Attachment monitor timed out'); 
            })
        ]);
    }
}
```

**DNR Rule Example:**

The system creates dynamic rules to intercept responses:

```javascript
_addDNRRule: async function(tabId, ruleId) {
    const rule = {
        id: ruleId,
        priority: 1,
        condition: {
            tabIds: [tabId],
            resourceTypes: ['main_frame', 'sub_frame']
        },
        action: {
            type: 'allow' // Allow the request, but monitor response
        }
    };
    
    // Add listener for response headers
    // Check for:
    // - Content-Type: application/pdf
    // - Content-Disposition: attachment
    // - Successful redirect to PDF URL
}
```

---

## Data Transfer to Zotero

### 1. Metadata Preparation

**File:** `src/common/itemSaver_background.js`

```javascript
Zotero.ItemSaver.saveStandaloneAttachmentToZotero = async function(
    attachment, 
    sessionID, 
    tab
) {
    // Step 1: Fetch PDF as ArrayBuffer
    let arrayBuffer = await this._fetchAttachment(attachment, tab);

    // Step 2: Prepare metadata with RFC2047 encoding for non-ASCII characters
    let metadata = JSON.stringify({
        url: attachment.url,
        contentType: attachment.mimeType,
        title: this._rfc2047Encode(attachment.title),
    });

    // Step 3: Send to Zotero via connector API
    return Zotero.Connector.callMethod({
        method: "saveStandaloneAttachment",
        headers: {
            "Content-Type": `${attachment.mimeType}`,
            "X-Metadata": metadata
        },
        queryString: `sessionID=${sessionID}`,
        timeout: 60e3
    }, arrayBuffer);
}
```

**RFC2047 Encoding:**

For titles with non-ASCII characters (e.g., "Über den Wolken"):

```javascript
Zotero.ItemSaver._rfc2047Encode = function(str) {
    if (!str || !this._containsNonAsciiChars(str)) {
        return str;
    }
    
    const utf8Bytes = new TextEncoder().encode(str);
    let encoded = '';
    for (let byte of utf8Bytes) {
        if (byte === 32) { // space
            encoded += '_';
        } else if (byte >= 33 && byte <= 126 && 
                   byte !== 61 && byte !== 63 && byte !== 95) {
            encoded += String.fromCharCode(byte);
        } else {
            encoded += '=' + byte.toString(16).toUpperCase().padStart(2, '0');
        }
    }
    
    // Returns: =?UTF-8?Q?encoded_text?=
    return `=?UTF-8?Q?${encoded}?=`;
}
```

### 2. HTTP Request to Zotero

**File:** `src/common/connector.js`

```javascript
Zotero.Connector.callMethod = function(options, body) {
    // Construct URL to Zotero's connector server
    const url = `http://localhost:23119/connector/${options.method}`;
    
    // Make HTTP POST request
    return Zotero.HTTP.request("POST", url, {
        headers: options.headers,
        body: body, // ArrayBuffer with PDF data
        timeout: options.timeout || 30000
    });
}
```

**Request Structure:**
- **Method:** POST
- **URL:** `http://localhost:23119/connector/saveStandaloneAttachment?sessionID=abc123`
- **Headers:**
  - `Content-Type: application/pdf`
  - `X-Metadata: {"url":"...","contentType":"application/pdf","title":"..."}`
- **Body:** PDF data as ArrayBuffer

### 3. Zotero Server Processing

The Zotero standalone client receives the request at its connector server:

1. **Parses metadata** from `X-Metadata` header
2. **Creates attachment item** in the library
3. **Saves PDF file** to Zotero's storage directory
4. **Returns success response** with item details

---

## Progress Tracking and UI Updates

### Progress Window System

**File:** `src/common/inject/pageSaving.js`

```javascript
async _saveAsStandaloneAttachment({ title=document.title } = {}) {
    const sessionID = this.sessionDetails.id;
    
    // Step 1: Create progress item
    let progressItem = {
        sessionID,
        id: 1,
        iconSrc: Zotero.ItemTypes.getImageSrc(`attachment-pdf`),
        title,
        progress: 0,
        itemType: Zotero.getString(`itemType_pdf`),
    };

    // Step 2: Show progress window
    Zotero.Messaging.sendMessage("progressWindow.show", [
        sessionID, 
        null, 
        false, 
        true
    ]);
    
    // Step 3: Initial progress update
    Zotero.Messaging.sendMessage(
        "progressWindow.itemProgress",
        progressItem
    );

    // Step 4: Perform download
    let standaloneAttachment = {
        url: document.location.toString(),
        mimeType: document.contentType,
        title,
        linkMode: "imported_url",
        referrer: document.referrer
    }

    try {
        await Zotero.ItemSaver.saveStandaloneAttachmentToZotero(
            standaloneAttachment, 
            sessionID
        )
        
        // Step 5: Mark session as created
        Zotero.Messaging.sendMessage(
            "progressWindow.sessionCreated", 
            { sessionID }
        );
        
        // Step 6: Update progress to 100%
        Zotero.Messaging.sendMessage(
            "progressWindow.itemProgress", 
            { ...progressItem, progress: 100 }
        );
        
    } catch (e) {
        // Step 7: Handle errors
        Zotero.Messaging.sendMessage(
            "progressWindow.error", 
            {
                sessionID,
                title,
                error: e.message
            }
        );
    }
}
```

**Progress States:**
1. **0%**: Download initiated
2. **0-100%**: Download in progress (for large files)
3. **100%**: Successfully saved to Zotero
4. **Error**: Display error message

---

## Key Components and Files

### Core Files

| File Path | Purpose |
|-----------|---------|
| `src/browserExt/webRequestIntercept.js` | Intercepts HTTP responses to detect PDFs in frames |
| `src/browserExt/background.js` | Main background script, handles UI updates and user actions |
| `src/common/inject/inject.jsx` | Injected into every webpage, initializes translation framework |
| `src/common/inject/pageSaving.js` | Orchestrates page and attachment saving from content script |
| `src/common/itemSaver.js` | Shared item saving logic |
| `src/common/itemSaver_background.js` | Background-side attachment fetching and bot bypass |
| `src/browserExt/browserAttachmentMonitor/browserAttachmentMonitor.js` | Intercepts PDF downloads during bot bypass |
| `src/browserExt/browserAttachmentMonitor/browserAttachmentMonitor.html` | HTML page for attachment monitor |
| `src/common/connector.js` | API for communicating with Zotero client |
| `src/common/translate.js` | Translation framework base classes |
| `src/common/translateWeb.js` | Web-specific translation logic |
| `src/common/messages.js` | Message passing definitions |
| `src/common/messaging.js` | Background message listener registration |

### Data Structures

**Attachment Object:**
```javascript
{
    url: "https://example.com/article.pdf",
    mimeType: "application/pdf",
    title: "Article Title",
    linkMode: "imported_url",
    referrer: "https://example.com/article-page",
    parentItem: "ABC123",  // For attachments to items
    id: 1                  // Local ID for progress tracking
}
```

**Progress Item:**
```javascript
{
    sessionID: "xyz789",
    id: 1,
    iconSrc: "chrome-extension://..../icons/attachment-pdf.png",
    title: "Article Title",
    progress: 50,
    itemType: "PDF"
}
```

---

## Complete Flow Diagram

```
┌───────────────────────────────────────────────────────────────────┐
│                    USER VISITS JOURNAL ARTICLE PAGE               │
└──────────────────────────┬────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 1: PDF DETECTION (webRequestIntercept.js)          │
├─────────────────────────────────────────────────────────────────┤
│  • browser.webRequest.onHeadersReceived listener               │
│  • Checks Content-Type header in all HTTP responses            │
│  • Detects: application/pdf or application/epub+zip            │
│  • Triggers: Zotero.Connector_Browser.onPDFFrame()            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 2: UI UPDATE (background.js)                       │
├─────────────────────────────────────────────────────────────────┤
│  • onPDFFrame() → onTranslators()                              │
│  • Updates tabInfo with isPDF: true                            │
│  • _updateExtensionUI(tab)                                     │
│  • Changes browser action icon to PDF icon                     │
│  • Updates tooltip: "Save to Zotero (PDF)"                     │
│  • Enables context menu item                                   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                  ┌────────┴────────┐
                  │  USER CLICKS    │
                  │  SAVE BUTTON    │
                  └────────┬────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 3: SAVE INITIATION (background.js)                 │
├─────────────────────────────────────────────────────────────────┤
│  • _browserAction(tab) triggered                               │
│  • Checks tabInfo.isPDF == true                                │
│  • Sends message: "saveAsWebpage" to content script            │
│  • Options: { snapshot: true }                                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 4: ROUTE DECISION (pageSaving.js)                  │
├─────────────────────────────────────────────────────────────────┤
│  • onSaveAsWebpage() receives message                          │
│  • Checks: document.contentType                                │
│  • If not text/* (i.e., application/pdf):                      │
│    → _saveAsStandaloneAttachment()                            │
│  • If text/html with translator:                              │
│    → _saveAsWebpage() with translator                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│    STEP 5: ATTACHMENT METADATA CREATION (pageSaving.js)         │
├─────────────────────────────────────────────────────────────────┤
│  • _saveAsStandaloneAttachment()                               │
│  • Creates session ID: Zotero.Utilities.randomString()         │
│  • Determines itemType from content type                       │
│  • Creates attachment object:                                  │
│    {                                                           │
│      url: document.location.toString(),                       │
│      mimeType: document.contentType,                          │
│      title: document.title,                                   │
│      linkMode: "imported_url",                                │
│      referrer: document.referrer                              │
│    }                                                           │
│  • Shows progress window (0%)                                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│       STEP 6: ATTACHMENT DOWNLOAD (itemSaver_background.js)     │
├─────────────────────────────────────────────────────────────────┤
│  • _fetchAttachment(attachment, tab)                           │
│  • Retrieves browser cookies (especially partitioned cookies)  │
│  • Sets headers:                                               │
│    - Cookie: session cookies                                  │
│    - Referrer: document.referrer                              │
│  • Executes: Zotero.HTTP.request("GET", url, options)         │
│  • Response type: arraybuffer                                 │
│  • Timeout: 60 seconds                                        │
│  • Validates Content-Type matches expectation                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                  ┌────────┴────────┐
                  │  Success?       │
                  └─────┬───────┬───┘
                   YES  │       │  NO (403/407/bot detection)
                        │       │
                        │       ▼
                        │  ┌─────────────────────────────────────────┐
                        │  │ STEP 6A: BOT BYPASS ATTEMPT             │
                        │  ├─────────────────────────────────────────┤
                        │  │ • Check if domain is whitelisted        │
                        │  │ • Method 1: Hidden iframe               │
                        │  │   - Inject invisible iframe             │
                        │  │   - Navigate to PDF URL in iframe       │
                        │  │   - JS executes, passes bot detection   │
                        │  │   - Monitor intercepts actual PDF       │
                        │  │   - Timeout: 5 seconds                  │
                        │  │ • If Method 1 fails → Method 2:         │
                        │  │   - Open popup window                   │
                        │  │   - User solves CAPTCHA                 │
                        │  │   - Monitor intercepts PDF download     │
                        │  │   - Timeout: 60 seconds                 │
                        │  └─────────────┬───────────────────────────┘
                        │                │
                        │                ▼
                        │  ┌─────────────────────────────────────────┐
                        │  │ STEP 6B: MONITOR INTERCEPTION           │
                        │  │ (browserAttachmentMonitor.js)           │
                        │  ├─────────────────────────────────────────┤
                        │  │ • Adds DNR rule (Chrome) or             │
                        │  │   webRequest listener (Firefox)         │
                        │  │ • Monitors for:                         │
                        │  │   - Content-Type: application/pdf       │
                        │  │   - Content-Disposition: attachment     │
                        │  │   - Successful redirect to PDF URL      │
                        │  │ • Captures final PDF URL                │
                        │  │ • Returns to _fetchAttachment()         │
                        │  └─────────────┬───────────────────────────┘
                        │                │
                        └────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│      STEP 7: METADATA ENCODING (itemSaver_background.js)        │
├─────────────────────────────────────────────────────────────────┤
│  • RFC2047 encode non-ASCII characters in title                │
│  • Example: "Über" → "=?UTF-8?Q?=C3=9Cber?="                  │
│  • Create metadata JSON:                                       │
│    {                                                           │
│      url: "https://...",                                      │
│      contentType: "application/pdf",                          │
│      title: "=?UTF-8?Q?encoded_title?="                      │
│    }                                                           │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│      STEP 8: SEND TO ZOTERO (connector.js)                      │
├─────────────────────────────────────────────────────────────────┤
│  • Zotero.Connector.callMethod({                               │
│      method: "saveStandaloneAttachment",                       │
│      headers: {                                                │
│        "Content-Type": "application/pdf",                      │
│        "X-Metadata": "{...metadata JSON...}"                  │
│      },                                                        │
│      queryString: "sessionID=xyz789"                          │
│    }, arrayBuffer)                                             │
│  • POST to: http://localhost:23119/connector/                 │
│              saveStandaloneAttachment?sessionID=xyz789         │
│  • Body: ArrayBuffer containing PDF binary data               │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 9: ZOTERO PROCESSES REQUEST                        │
├─────────────────────────────────────────────────────────────────┤
│  • Zotero Connector Server (port 23119)                       │
│  • Parses X-Metadata header                                   │
│  • Creates attachment item in library                         │
│  • Generates unique storage path                              │
│  • Writes PDF to: ~/Zotero/storage/ABC123XYZ/file.pdf        │
│  • Returns success response:                                  │
│    { success: true, canRecognize: true }                      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│        STEP 10: PROGRESS UPDATE (pageSaving.js)                 │
├─────────────────────────────────────────────────────────────────┤
│  • Receives success response                                   │
│  • Sends message: "progressWindow.sessionCreated"              │
│  • Updates progress: 100%                                      │
│  • If canRecognize == true:                                    │
│    - Triggers PDF metadata recognition                         │
│    - Extracts: DOI, title, authors from PDF                   │
│    - Creates proper bibliographic item                         │
│    - Attaches PDF to the item                                 │
│  • Shows success message in progress window                    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      COMPLETE ✓                                 │
│  PDF saved to Zotero library with metadata                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Technical Details

### Message Passing Architecture

The Zotero Connector uses a sophisticated message passing system to communicate between content scripts (injected into webpages) and the background process.

**File:** `src/common/messages.js`

```javascript
// Defines which methods can be called across contexts
Zotero.Messaging.MESSAGES = {
    "saveAsWebpage": true,  // Returns response
    "progressWindow.show": false,  // No response
    "progressWindow.itemProgress": false,
    "ItemSaver.saveStandaloneAttachmentToZotero": true
}
```

**Content Script → Background:**

```javascript
// In inject.jsx or pageSaving.js
Zotero.Messaging.sendMessage("saveAsWebpage", {
    title: "Article Title",
    snapshot: true
});
```

**Background → Content Script:**

```javascript
// In background.js
Zotero.Messaging.sendMessage("progressWindow.itemProgress", {
    sessionID: "abc123",
    progress: 50
}, tab);
```

### Cookie Handling

Modern browsers partition cookies for security. The connector must handle both regular and partitioned cookies:

```javascript
// Get cookies with partitionKey (Chrome 118+)
let cookies = await browser.cookies.getAll({
    url: attachment.url,
    partitionKey: {},
});

// Fallback for older browsers
catch (e) {
    cookies = await browser.cookies.getAll({
        url: attachment.url,
    });
}

// Filter for partitioned cookies (e.g., Cloudflare clearance)
cookies = cookies.filter(c => c.partitionKey);
```

**Why this matters:**
- Cloudflare uses partitioned cookies for bot detection
- Without these cookies, download may fail with 403 error
- Must send cookies in Cookie header for XHR requests

### Content Type Detection

The connector uses multiple methods to determine if content is a PDF:

1. **HTTP Header:** `Content-Type: application/pdf`
2. **Document API:** `document.contentType === 'application/pdf'`
3. **URL Extension:** `guessAttachmentMimeType(url)` checks file extension
4. **Lenient Matching:** Accepts `application/octet-stream` for PDFs

```javascript
// Validate received content type
if (attachment.mimeType.toLowerCase() === receivedMimeType.toLowerCase()) {
    return xhr.response;
}

// Accept octet-stream for PDFs (some hosts)
const expectedPDFIsOctetStream = 
    attachment.mimeType.toLowerCase() === 'application/pdf'
    && receivedMimeType.toLowerCase() === 'application/octet-stream'

if (expectedPDFIsOctetStream) {
    return xhr.response;
}
```

### Error Handling

The system implements multiple fallback layers:

```
Direct HTTP Request
    ↓ (fails with 403/407)
Hidden Iframe Method
    ↓ (timeout after 5s)
Popup Window Method
    ↓ (user interaction, timeout 60s)
Error Display to User
```

### Session Management

Each save operation gets a unique session ID:

```javascript
const sessionID = Zotero.Utilities.randomString();
this.sessionDetails = {
    id: sessionID,
    url: document.location.href,
    translatorID,
    saveOptions
};
```

**Purpose:**
- Track progress across multiple items
- Associate attachments with parent items
- Enable multiple concurrent saves
- Allow progress updates via messaging

### Translator Integration

When saving from a translator (not standalone PDF):

```javascript
// Translator returns items with attachments
{
    itemType: "journalArticle",
    title: "Article Title",
    authors: [...],
    attachments: [
        {
            url: "https://example.com/article.pdf",
            mimeType: "application/pdf",
            title: "Full Text PDF"
        }
    ]
}
```

The system:
1. Filters for primary attachment types (PDF, EPUB)
2. Downloads attachments in parallel
3. Marks primary attachment for Zotero
4. Reports progress for each attachment

---

## Conclusion

The Zotero Connector's PDF extraction system is a sophisticated, multi-layered architecture that:

1. **Detects PDFs** through web request interception and content type analysis
2. **Manages user interaction** via browser action buttons and context menus
3. **Downloads PDFs** with proper authentication (cookies, referrers)
4. **Bypasses bot protection** using hidden iframes and user-assisted popup windows
5. **Transfers data** to Zotero with proper encoding and metadata
6. **Provides feedback** through a real-time progress tracking system

The system handles edge cases including:
- CAPTCHA and bot detection systems
- Partitioned cookies for modern browsers
- Non-ASCII characters in filenames
- Multiple concurrent downloads
- Content type validation and fallbacks
- Network timeouts and errors

This architecture enables Zotero users to seamlessly save PDFs from journal articles across thousands of different publisher platforms, even when faced with anti-bot measures and authentication requirements.

---

## Appendix: Key Code Snippets

### Complete PDF Save Flow (Simplified)

```javascript
// 1. Detection
webRequestIntercept.js: onHeadersReceived() 
  → detects application/pdf

// 2. UI Update  
background.js: onPDFFrame()
  → updates button icon

// 3. User Action
background.js: _browserAction()
  → sends "saveAsWebpage" message

// 4. Content Script Processing
pageSaving.js: onSaveAsWebpage()
  → calls _saveAsStandaloneAttachment()
  → creates attachment metadata
  → shows progress window

// 5. Background Download
itemSaver_background.js: saveStandaloneAttachmentToZotero()
  → _fetchAttachment()
    → HTTP.request() with cookies
    → [if fails] → _passJSBotDetectionViaHiddenIframe()
    → [if fails] → _passJSBotDetectionViaWindowPrompt()
  → returns ArrayBuffer

// 6. Send to Zotero
itemSaver_background.js: 
  → RFC2047 encode title
  → Connector.callMethod("saveStandaloneAttachment", metadata, arrayBuffer)
  → POST to localhost:23119

// 7. Progress Update
pageSaving.js:
  → progressWindow.itemProgress (100%)
  → [if canRecognize] → extract metadata from PDF

// 8. Complete
  → PDF in Zotero library
```

### Error Handling Example

```javascript
try {
    // Direct download attempt
    let xhr = await Zotero.HTTP.request("GET", url, options);
    return xhr.response;
} catch (e) {
    if (e.status === 404) {
        throw new Error("PDF not found");
    }
    
    // Attempt bot bypass
    try {
        // Hidden iframe method
        let pdfURL = await _passJSBotDetectionViaHiddenIframe(url, tab);
        return await _fetchAttachment({ url: pdfURL });
    } catch (e2) {
        // Popup window method (requires user interaction)
        let pdfURL = await _passJSBotDetectionViaWindowPrompt(url, tab);
        return await _fetchAttachment({ url: pdfURL });
    }
}
```

---

**Document Version:** 1.0  
**Last Updated:** 2026-02-09  
**Author:** Automated Analysis of Zotero Connectors Repository
