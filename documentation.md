# 🔑 Key Page — Full Documentation

> **Grandfuscator Panel** · Feature  
> Protect your Lua scripts with a key validation system.

---

## Table of Contents

1. [Overview](#overview)
2. [How It Works](#how-it-works)
3. [Setup Guide](#setup-guide)
   - [Step 1 — Access Your Key Manager](#step-1--access-your-key-manager)
   - [Step 2 — Configure Your Page](#step-2--configure-your-page)
   - [Step 3 — Share Your Public Page](#step-3--share-your-public-page)
   - [Step 4 — Add Keys Manually](#step-4--add-keys-manually)
4. [API Reference](#api-reference)
5. [Integrating the Key Check into Your Script](#integrating-the-key-check-into-your-script)
   - [Minimal Example](#minimal-example)
   - [Full Example with Periodic Re-check](#full-example-with-periodic-re-check)
6. [Key Format](#key-format)
7. [Configuration Options](#configuration-options)
8. [Free Tier Limitations](#free-tier-limitations)
9. [Tier Comparison](#tier-comparison)
10. [FAQ](#faq)

---

## Overview

The Key Page system lets you protect your Lua scripts by requiring users to hold a valid key before your script runs. As a **Free tier** user, you get your own:

- A **public page** where users complete Linkvertise tasks and generate their key
- A **private configuration page** where you manage all settings and keys
- A **REST API endpoint** your script calls to validate a key in real-time

All key data is stored privately in Firebase — only you can see your own keys.

---

## How It Works

```
User opens your Public Page
        │
        ▼
User completes N Linkvertise tasks  (N = set by you, 1–20)
        │
        ▼
User clicks "Generate My Key"
        │
        ▼
System creates a GRAND-KEY-XXXXXXXX  (valid for X hours, set by you)
        │
        ▼
User pastes the key into your script
        │
        ▼
Script calls the API → API checks Firebase → returns { valid: true/false }
        │
        ▼
Script continues or stops based on the result
```

---

## Setup Guide

### Step 1 — Access Your Key Manager

Log in to the panel with your Free (or above) account, then open the sidebar and click **My Key Manager**.

You will see two important URLs on the dashboard:

| URL | Purpose |
|-----|---------|
| `?view=freekeypage&id=<username>` | **Public Page** — share this with your users |
| `?view=freekeymanager` | **Configuration Page** — keep this private |

---

### Step 2 — Configure Your Page

In the **Configuration** tab, fill in the following fields:

#### Linkverse ID *(required)*
Your Linkvertise publisher user ID. This is the number found in your Linkvertise dashboard URL, for example:
```
https://linkvertise.com/publisher/offers
                                 └─ your user ID is in the offer link
```
When a user clicks **Get Key**, the panel sends them through your Linkvertise offer link. You earn ad revenue for every completion.

#### Linkvertise Anti-Bypass Token *(optional but recommended)*
Found in your Linkvertise dashboard under **Settings → Anti Bypassing**.  
When set, the panel verifies with Linkvertise that the user actually completed the page (not bypassed it with a tool). This is handled entirely server-side — you do not need to do anything in your Lua script.

#### Tasks Required
A slider from **1 to 20**. The user must complete this many Linkvertise tasks before they can generate a key. Higher values = more ad revenue per key, but more friction for the user.

#### Key Duration
How long each generated key stays valid. Choose from preset values (1h, 6h, 12h, 24h, 48h, 72h) or enter a custom number of hours.

> **Example:** If you set 3 tasks and 24h duration, a user must pass through Linkvertise 3 times, then receives a key that expires after 24 hours.

Click **Save Configuration** when done.

---

### Step 3 — Share Your Public Page

Copy your **Public Page URL** from the dashboard and share it wherever your users can find it — Discord server, GitHub README, or your script's UI.

The public page handles everything automatically:
- Tracks progress per visitor session
- Sends users through Linkvertise
- Generates a `GRAND-KEY-XXXXXXXX` key on completion
- Lets users validate an existing key

---

### Step 4 — Add Keys Manually

In the **Keys** tab you can manually add keys — useful for giving keys to specific people without requiring them to complete Linkvertise tasks.

| Field | Required | Description |
|-------|----------|-------------|
| Key Value | ✅ | The key string (e.g. `GRAND-KEY-ABCD1234`) |
| Expiry Date | ✅ | Date the key becomes invalid |
| Note | optional | Internal label (e.g. "Given to user123") |

> Manual keys are also validated by the same API. The user experience is identical.

---

## API Reference

### Endpoint

```
GET https://www.grandfuscator.my.id/api/grandfuscator-keyvalidation
```

### Query Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `name` | ✅ | Your panel username (case-sensitive) |
| `checkkey` | ✅ | The key to validate |

### Example Request

```
https://www.grandfuscator.my.id/api/grandfuscator-keyvalidation?name=yourusername&checkkey=GRAND-KEY-ABCD1234
```

### Response — Valid Key

```json
{
  "valid": true,
  "message": "Key is valid",
  "expiresAt": 1780000000
}
```

### Response — Invalid / Expired Key

```json
{
  "valid": false,
  "message": "Key not found"
}
```

```json
{
  "valid": false,
  "message": "Key has expired"
}
```

```json
{
  "valid": false,
  "message": "Missing parameter: name"
}
```

> **Note:** `expiresAt` is a Unix timestamp (seconds). Convert with `os.date()` in Lua or `new Date(ts * 1000)` in JavaScript.

---

## Integrating the Key Check into Your Script

Place the key check code at the **very top** of your script, before any other logic. The script should stop immediately if the key is invalid.

### Minimal Example

```lua
local KEY_API       = "https://www.grandfuscator.my.id/api/grandfuscator-keyvalidation"
local YOUR_USERNAME = "yourusername"   -- ← CHANGE THIS to your panel username

-- File paths where the key is saved on the device
-- Two paths for compatibility with different GrowLauncher versions
local paths = {
    "/storage/emulated/0/Android/media/launcher.powerkuy.growlauncher/.mykey.lua",
    "/storage/emulated/0/Android/data/launcher.powerkuy.growlauncher/files/.mykey.lua"
}

-- Saves the key to disk so the user doesn't need to re-enter it next time
function saveKey(k)
    for _, p in ipairs(paths) do
        local f = io.open(p, "w")
        if f then f:write("key=" .. k); f:close(); return end
    end
end

-- Reads a previously saved key from disk
function loadSavedKey()
    for _, p in ipairs(paths) do
        local f = io.open(p, "r")
        if f then
            local s = f:read("*all"); f:close()
            local k = s:match("key=(.+)")
            if k then return k:match("^%s*(.-)%s*$") end
        end
    end
    return nil
end

-- Calls the API and returns true/false + message
function checkKey(key)
    if not key or key == "" then return false, "Key is empty" end
    local url = KEY_API .. "?name=" .. YOUR_USERNAME .. "&checkkey=" .. key
    local raw, err = fetch(url)
    if not raw then return false, "Network error: " .. tostring(err) end
    if raw:find('"valid":true') then return true, "OK" end
    return false, (raw:match('"message":"([^"]+)"') or "Invalid key")
end

-- ── Run the check ───────────────────────────────────────────────────────────
local myKey = loadSavedKey() or ""
local ok, msg = checkKey(myKey)

if not ok then
    LogToConsole("[Script] Key check failed: " .. msg)
    return  -- stop the script
end

saveKey(myKey)
LogToConsole("[Script] Key OK! Starting...")

-- ── Your script logic starts here ───────────────────────────────────────────
```

---

### Full Example with Periodic Re-check

This version re-validates the key every 10 minutes while the script is running. If the key is revoked or expires mid-session, the script stops automatically.

```lua
local KEY_API       = "https://www.grandfuscator.my.id/api/grandfuscator-keyvalidation"
local YOUR_USERNAME = "yourusername"   -- ← CHANGE THIS
local KEY_RECHECK   = 10 * 60         -- re-check interval in seconds (10 minutes)

local myKey = ""

local paths = {
    "/storage/emulated/0/Android/media/launcher.powerkuy.growlauncher/.mykey.lua",
    "/storage/emulated/0/Android/data/launcher.powerkuy.growlauncher/files/.mykey.lua"
}

function validateKey(key)
    if not key or key == "" then return false, "Key is empty" end
    local url = KEY_API .. "?name=" .. YOUR_USERNAME .. "&checkkey=" .. key
    local raw, err = fetch(url)
    if not raw then return false, "Network error: " .. tostring(err) end
    if raw:find('"valid":true') then return true, "OK" end
    return false, (raw:match('"message":"([^"]+)"') or "Invalid key")
end

function loadSavedKey()
    for _, p in ipairs(paths) do
        local f = io.open(p, "r")
        if f then
            local s = f:read("*all"); f:close()
            local k = s:match("key=(.+)")
            if k then return k:match("^%s*(.-)%s*$") end
        end
    end
    return nil
end

function saveKey(k)
    for _, p in ipairs(paths) do
        local f = io.open(p, "w")
        if f then f:write("key=" .. k); f:close(); return end
    end
end

-- ── Initial check ───────────────────────────────────────────────────────────
myKey = loadSavedKey() or ""
local ok, msg = validateKey(myKey)

if not ok then
    LogToConsole("Key validation failed: " .. msg)
    sendDialog({
        title   = "Key Required",
        message = "Your key is invalid or expired:\n" .. msg ..
                  "\n\nGet a key at: grandfuscator.my.id"
    })
    return  -- stop the script
end

saveKey(myKey)
LogToConsole("Key OK, starting script...")

-- ── Periodic re-check (runs in background) ──────────────────────────────────
local lastCheck = os.time()
runThread(function()
    while true do
        Sleep(60000)  -- check every 60 seconds if interval has passed
        if os.time() - lastCheck >= KEY_RECHECK then
            lastCheck = os.time()
            local valid, vmsg = validateKey(myKey)
            if not valid then
                LogToConsole("Key revoked: " .. vmsg)
                stopmyscript()  -- ← replace with your actual stop function
                return
            end
        end
    end
end)

-- ── Your script logic starts here ───────────────────────────────────────────
-- startFarming() or whatever your script does
```

> **Important:** Replace `stopmyscript()` on the periodic re-check section with whatever function your script uses to stop itself (e.g. `stopFarming()`, `stopBot()`, etc.).

---

## Key Format

Keys generated automatically through the public page always follow this format:

```
GRAND-KEY-XXXXXXXX
```

Where `XXXXXXXX` is a random 8-character alphanumeric string (uppercase).

**Examples:**
```
GRAND-KEY-A3F9K2PQ
GRAND-KEY-ZX17BCDE
GRAND-KEY-90MNRTYV
```

> The `GRAND-KEY-` prefix is **fixed** for the Free tier and cannot be changed. VVIP tier users can set a custom prefix.

---

## Configuration Options

| Setting | Default | Range | Description |
|---------|---------|-------|-------------|
| Linkverse ID | — | — | Your Linkvertise publisher user ID. Required. |
| Anti-Bypass Token | — | — | From Linkvertise → Settings → Anti Bypassing. Optional. |
| Tasks Required | 3 | 1 – 20 | How many Linkvertise steps the user must complete. |
| Key Duration | 24h | 1h – ∞ | How long each generated key remains valid. |

---

## Free Tier Limitations

| Feature | Free Tier | VVIP Tier |
|---------|-----------|-----------|
| Key validation API | ✅ | ✅ |
| Public key page | ✅ | ✅ |
| Manual key management | ✅ | ✅ |
| Linkvertise task system | ✅ | ✅ |
| Anti-bypass token | ✅ | ✅ |
| Custom key prefix | ❌ Fixed: `GRAND-KEY-` | ✅ Custom prefix |
| Discord ID validation | ❌ Not available | ✅ Available |
| Key prefix branding | ❌ | ✅ |

---

## Tier Comparison

```
Free Tier                         VVIP Tier
─────────────────────────────     ─────────────────────────────────────
Key prefix: GRAND-KEY-XXXX        Key prefix: YOURCOOLNAME-XXXX (custom)
Discord ID check: NO              Discord ID check: YES (optional)
Public page: YES                  Public page: YES
API validation: YES               API validation: YES
Manual keys: YES                  Manual keys: YES + Discord ID per key
```

---

## FAQ

**Q: Where do users get their key?**  
A: Share your **Public Page URL** (`?view=freekeypage&id=yourusername`). Users go there, complete the Linkvertise tasks you configured, and receive their key automatically.

**Q: Do I need to manually create a key for every user?**  
A: No. The public page auto-generates keys for users who complete the tasks. You only need to add keys manually if you want to give someone a key without requiring Linkvertise.

**Q: What happens when a key expires?**  
A: The API returns `{ "valid": false, "message": "Key has expired" }`. Your script receives this and should stop. The user needs to visit the public page again to get a new key.

**Q: Can users share their key with others?**  
A: With the Free tier, keys are not tied to a specific device or account, so technically yes. If you want to prevent sharing, upgrade to **VVIP** and enable the Discord ID validation feature, which ties each key to a specific Discord account.

**Q: Is the key data secure? Can anyone else see my keys?**  
A: Your keys are stored in a private Firebase path (`free_key_pages/yourusername/keys`). The panel Owner can only see the count of your keys, not their actual values. Nobody else has read access to your key data.

**Q: What if a user loses their key?**  
A: The public page saves key progress per browser session. If a user already generated a valid key that hasn't expired, the page will show it again when they click "Generate". If the session is lost, they need to complete the tasks again.

**Q: Can the admin disable my key page?**  
A: Yes. The panel Owner can disable any key server page if it violates the rules or causes issues. Users visiting a disabled page will see a "Service Unavailable" message. Your key data remains intact and the page can be re-enabled.

**Q: Does the Linkvertise anti-bypass token affect my Lua script?**  
A: No. The anti-bypass token is only used between the web panel and Linkvertise's API to verify the user completed the page. Your Lua script only ever calls `grandfuscator-keyvalidation` — it has nothing to do with the token.

**Q: What is `loadSavedKey()` for?**  
A: It reads a small file saved on the user's device that stores their key from a previous session. This means users only need to paste their key once — the script remembers it for next time. The file is saved at two possible paths to support both GrowLauncher v6 and v7 directory structures.

---

*Grandfuscator*  
*For support or questions, visit the panel or contact the server owner.*
