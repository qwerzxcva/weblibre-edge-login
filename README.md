# WebLibre Edge Account Login

A WebExtension that lets a Microsoft account sign-in complete on a Gecko engine
(WebLibre, Firefox for Android, GeckoView) by presenting Microsoft's identity
pages as Microsoft Edge.

## Why this is needed

Microsoft's account endpoints sniff the browser and answer anything they do not
recognise with a dead end:

- `https://account.live.com/error.aspx?errcode=1047` — *"Browser not supported.
  To sign in to Microsoft account, try upgrading your browser."*
- `https://login.live.com` — loops back to the password prompt instead of
  completing the flow.

The decision is made from the **User-Agent header**, before any page script
runs, which is why a normal content script cannot fix it on its own.

## What it does

On Microsoft's identity domains only, it makes the browser look like Microsoft
Edge on Android, consistently across every surface a page can check:

| Surface | Set to |
|---|---|
| `User-Agent` request header | Edge on Android (`…Chrome/120… EdgA/120…`) |
| `Sec-CH-UA`, `Sec-CH-UA-Mobile`, `Sec-CH-UA-Platform` | `Microsoft Edge`, `?1`, `Android` |
| `navigator.userAgent`, `appVersion`, `vendor`, `productSub` | matching values |
| `navigator.userAgentData` (brands, mobile, platform, `getHighEntropyValues`) | `Microsoft Edge` brand list |
| `window.chrome` | provided, because Chromium has it |

The header rewrite is `background.js`; the JavaScript surfaces are
`content.js`. Both are needed: a page told it is Edge by the header that then
reads `navigator.userAgent` and finds Firefox has been given a reason to abort
the flow, and the reverse mismatch is just as visible.

`EdgA/` is the Android Edge token — desktop Edge uses `Edg/`, iOS uses
`EdgiOS/`.

## Scoped to Microsoft domains

Nothing outside these is touched, so the rest of the web still sees the real
browser:

```
*://*.live.com/*              *://*.office.com/*
*://*.microsoft.com/*         *://*.office365.com/*
*://*.microsoftonline.com/*   *://*.sharepoint.com/*
*://*.microsoft365.com/*      *://login.windows.net/*
*://*.bing.com/*
```

## Installing

### As a built-in extension in WebLibre

Copy this directory into WebLibre's bundled extensions and register it:

```
packages/flutter_mozilla_components/android/src/main/assets/extensions/edge_login/
```

```kotlin
BuiltInWebExtensionController(
    "edge-login@weblibre.eu",
    "resource://android/assets/extensions/edge_login/",
    "edgeLogin",
).install(engine)
```

### As an unsigned add-on

WebLibre can install unsigned extensions when *Settings → Extensions → Allow
unsigned extensions* is on. Zip the contents of this directory (the manifest
must be at the archive root) and install the resulting `.xpi`.

## Verifying

1. Open `https://account.microsoft.com` and sign in. The sign-in form should
   appear instead of the "Browser not supported" page.
2. Check the identity is coherent: `navigator.userAgent` ends with `EdgA/120…`,
   `navigator.userAgentData.brands` lists `Microsoft Edge`, and `window.chrome`
   exists.

## Limitations

- **This only fixes browser sniffing.** If the sign-in still fails, the cause is
  something else — most often cookies. Microsoft's flow redirects across
  `login.live.com` → `account.live.com` → `login.microsoftonline.com`, so a
  browser that blocks third-party storage or isolates cookies per site can break
  it even with a perfect User-Agent. Allow cookies/storage for the Microsoft
  domains in that case.
- **It is a User-Agent override, so it will fight any other extension that
  spoofs the User-Agent on the same domains.** Turn the other one off for these
  sites, or accept that the last one to write the header wins.
- The Edge version is pinned in `content.js` and `background.js`. Bump
  `EDGE_VERSION` in both when it goes stale.
