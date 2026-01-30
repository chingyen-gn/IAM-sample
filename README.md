# Braze In-App Message - Cross-Origin Isolation Issue Reproducer

This is a minimal reproducer demonstrating an issue where Braze In-App Message images are blocked when Cross-Origin Isolation is enabled.

## Problem

Our app enables cross-origin isolation by setting these HTTP headers:

```
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Opener-Policy: same-origin
```

With these headers, the browser blocks any cross-origin resources that don't explicitly opt in via either CORS or the `Cross-Origin-Resource-Policy` header.

Braze CDN doesn't currently include this header, which causes images in In-App Messages to be blocked.

## SDK Version

- `@braze/web-sdk`: **6.3.0**

## Prerequisites

- Node.js
- OpenSSL (optional, only for HTTPS)

## Setup

### 1. Install Dependencies

```bash
npm install
```

This installs `@braze/web-sdk` version 6.3.0 locally.

### 2. Generate SSL Certificates (one-time)

```bash
npm run generate-certs
```

### 3. Start the Server

**Option A: With Cross-Origin Isolation (reproduces the issue)**

```bash
npm start
```

Server starts at `https://localhost:3000` with COEP/COOP headers enabled. Images will be blocked.

**Option B: Without Cross-Origin Isolation (for comparison)**

```bash
npm run start:no-coi
```

Server starts at `https://localhost:3000` without COEP/COOP headers. Images will load normally.

Accept the self-signed certificate warning in your browser.

### 4. Configure Braze API Key

Open `https://localhost:3000` in your browser.

Enter your Braze API key and SDK endpoint in the configuration form.

## Verifying Cross-Origin Isolation

Open the browser DevTools console. You should see:

```
Cross-Origin Isolated: true
```

If you see `false`, the headers are not being applied correctly.

## Reproducing the Issue

1. Enter your Braze API key and SDK endpoint, then click **Initialize Braze SDK**
2. In your Braze dashboard, create a test In-App Message campaign:
   - Include an image in the message
   - Target the user ID you entered (default: `test-user-123`)
   - Set trigger to either:
     - **Session Start** - then click **Trigger New Session** button
     - **Custom Event** named `test_iam_trigger` - then click **Trigger Custom Event** button
3. Trigger the IAM using one of the buttons:
   - **Trigger New Session**: Calls `braze.openSession()` to start a new session
   - **Trigger Custom Event**: Calls `braze.logCustomEvent("test_iam_trigger")`
4. Observe in the Network tab that image requests to Braze CDN are blocked
5. Check the console for CORP-related errors

## Expected Browser Error

```
net::ERR_BLOCKED_BY_RESPONSE.NotSameOriginAfterDefaultedToSameOriginByCoep
```

## Requested Fix

Either of these solutions would resolve the issue:

1. **Add CORP header to CDN**: Add `Cross-Origin-Resource-Policy: cross-origin` header to Braze CDN responses
2. **Use credentialless iframe**: Set the `credentialless` attribute on the iframe wrapping In-App Messages

## File Structure

```
IAM-sample/
├── README.md           # This file
├── package.json        # Node.js dependencies (@braze/web-sdk 6.3.0)
├── serve.json          # Config for npx serve with COEP/COOP headers
├── serve-no-coi.json   # Config for npx serve without COEP/COOP headers
├── index.html          # Minimal Braze SDK integration
├── node_modules/       # Dependencies (after npm install)
├── cert.pem            # SSL certificate (generated, for HTTPS)
└── key.pem             # SSL private key (generated, for HTTPS)
```
