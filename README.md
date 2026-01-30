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
- OpenSSL (for generating self-signed certificates)

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

## Reproducing the Issue

### Quick Test (no Braze dashboard required)

1. Start the server with COI enabled: `npm start`
2. Open `https://localhost:3000` and accept the certificate warning
3. Verify the status panel shows **Cross-Origin Isolated: true**
4. Enter your Braze API key and SDK endpoint, then click **Initialize Braze SDK**
5. Click **Show Mock IAM (with image)** to display a test modal with a Braze CDN image
6. Observe in the Network tab that the image request is blocked
7. The modal will appear but the image will be missing

### Full Test (with Braze dashboard)

1. Follow steps 1-4 above
2. In your Braze dashboard, create a test In-App Message campaign:
   - Include an image in the message
   - Target the user ID you entered (default: `test-user-123`)
   - Set trigger to **Session Start** or **Custom Event** named `test_iam_trigger`
3. Trigger the IAM using one of the buttons:
   - **Trigger New Session**: Calls `braze.changeUser()` and `braze.openSession()`
   - **Trigger Custom Event**: Calls `braze.logCustomEvent("test_iam_trigger")`
4. Observe the same blocking behavior

### Expected Browser Error

```
net::ERR_BLOCKED_BY_RESPONSE.NotSameOriginAfterDefaultedToSameOriginByCoep
```

## Comparison Test (without COI)

To verify that images load correctly when Cross-Origin Isolation is disabled:

1. Stop the current server (Ctrl+C)
2. Start the server without COI: `npm run start:no-coi`
3. Open `https://localhost:3000` (you may need to clear site data or use incognito)
4. Verify the status panel shows **Cross-Origin Isolated: false**
5. Initialize the SDK and click **Show Mock IAM (with image)**
6. The image should load successfully

This confirms that the issue is specifically caused by COEP/COOP headers.

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
