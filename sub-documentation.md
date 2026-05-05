# 🔑 Key Validation Documentation

> Protect your Lua scripts with a key validation.

---


## 📦 Requirements

Your script must run in an environment that provides these functions:

| Function | Purpose |
|----------|---------|
| `fetch(url)` | HTTP GET request, returns string response |
| `Sleep(ms)` | Pause execution in milliseconds |
| `runThread(func)` | Run a function in a separate thread |
| `LogToConsole(msg)` | Print log message to console |
| `sendDialog(tbl)` | Display a dialog to the user |

> These functions are typically available in executors like **GrowLauncher v6/v7**.

---

## 🧱 System Architecture

The key system consists of 3 components:

```

1. Local Storage   — Save key so users don't re-enter it
2. API Validation  — Ask the server if the key is valid
3. Periodic Recheck — Ensure key hasn't expired mid-session

```

---

## 🔧 Step 1 — Define Constants

```lua
local KEY_API       = "https://www.grandfuscator.my.id/api/grandfuscator-keyvalidation"
local YOUR_USERNAME = "yourusername"   -- REPLACE with your panel username
local KEY_RECHECK   = 10 * 60          -- recheck every 10 minutes
```

---

🔧 Step 2 — Local Key Read/Write Functions

Store the key on the user's device so they don't need to paste it every time.

```lua
local paths = {
    -- GrowLauncher v6+
    "/storage/emulated/0/Android/media/launcher.powerkuy.growlauncher/.mykey.lua",
    -- GrowLauncher v7+
    "/storage/emulated/0/Android/data/launcher.powerkuy.growlauncher/files/.mykey.lua"
}

function loadSavedKey()
    for _, p in ipairs(paths) do
        local f = io.open(p, "r")
        if f then
            local content = f:read("*all")
            f:close()
            local key = content:match("key=(.+)")
            if key then
                return key:match("^%s*(.-)%s*$")  -- trim whitespace
            end
        end
    end
    return nil
end

function saveKey(k)
    for _, p in ipairs(paths) do
        local f = io.open(p, "w")
        if f then
            f:write("key=" .. k)
            f:close()
            return true
        end
    end
    return false
end
```

---

🔧 Step 3 — API Validation Function

```lua
function validateKey(key)
    -- Check if key is empty
    if not key or key == "" then
        return false, "Key is empty"
    end

    -- Call the API
    local url = KEY_API .. "?name=" .. YOUR_USERNAME .. "&checkkey=" .. key
    local response, errorMsg = fetch(url)

    -- Check for network errors
    if not response then
        return false, "Connection failed: " .. tostring(errorMsg)
    end

    -- Check response
    if response:find('"valid":true') then
        return true, "Valid"
    end

    -- Extract error message from JSON response
    local message = response:match('"message":"([^"]+)"') or "Invalid key"
    return false, message
end
```

---

🔧 Step 4 — Initial Validation (Place at the Top of Your Script)

This code runs the key check when the script first starts:

```lua
-- Read key from local storage
local myKey = loadSavedKey() or ""

-- Validate
local ok, msg = validateKey(myKey)

if not ok then
    -- Key not valid → stop the script
    LogToConsole("[Key] " .. msg)
    sendDialog({
        title   = "⚠️ Key Required",
        message = "Your key is invalid or expired.\n\n"
               .. "Reason: " .. msg .. "\n\n"
               .. "Get a new key at:\ngrandfuscator.my.id"
    })
    return  -- STOP script here
end

-- Key is valid → save it & continue
saveKey(myKey)
LogToConsole("[Key] Valid! Starting script...")
```

---

🔧 Step 5 — Periodic Recheck (Optional but Recommended)

Automatically stops the script if the key is revoked or expires mid-session:

```lua
local lastCheck = os.time()

runThread(function()
    while true do
        Sleep(60000)  -- check every 60 seconds

        if os.time() - lastCheck >= KEY_RECHECK then
            lastCheck = os.time()

            local valid, vmsg = validateKey(myKey)
            if not valid then
                LogToConsole("[Key] REVOKED: " .. vmsg)
                sendDialog({
                    title   = "⛔ Key Expired",
                    message = "Your key is no longer valid.\n\n"
                           .. "Reason: " .. vmsg .. "\n\n"
                           .. "Please get a new key."
                })
                stopmyscript()  -- REPLACE with your script's stop function
                return
            end
        end
    end
end)
```

Important: Replace stopmyscript() with the actual function that stops your script.

---

📝 Complete Script (All Parts Combined)

```lua
local KEY_API       = "https://www.grandfuscator.my.id/api/grandfuscator-keyvalidation"
local YOUR_USERNAME = "yourusername"   -- ← REPLACE THIS
local KEY_RECHECK   = 10 * 60          -- recheck every 10 minutes

local myKey = ""

local paths = {
    "/storage/emulated/0/Android/media/launcher.powerkuy.growlauncher/.mykey.lua",
    "/storage/emulated/0/Android/data/launcher.powerkuy.growlauncher/files/.mykey.lua"
}

-- Read key function 
function loadSavedKey()
    for _, p in ipairs(paths) do
        local f = io.open(p, "r")
        if f then
            local content = f:read("*all")
            f:close()
            local key = content:match("key=(.+)")
            if key then
                return key:match("^%s*(.-)%s*$")
            end
        end
    end
    return nil
end

-- Save key function 
function saveKey(k)
    for _, p in ipairs(paths) do
        local f = io.open(p, "w")
        if f then
            f:write("key=" .. k)
            f:close()
            return true
        end
    end
    return false
end

-- Validation function
function validateKey(key)
    if not key or key == "" then
        return false, "Key is empty"
    end

    local url = KEY_API .. "?name=" .. YOUR_USERNAME .. "&checkkey=" .. key
    local response, errorMsg = fetch(url)

    if not response then
        return false, "Connection failed: " .. tostring(errorMsg)
    end

    if response:find('"valid":true') then
        return true, "Valid"
    end

    local message = response:match('"message":"([^"]+)"') or "Invalid key"
    return false, message
end

-- INITIAL VALIDATION 
myKey = loadSavedKey() or ""
local ok, msg = validateKey(myKey)

if not ok then
    LogToConsole("[Key] " .. msg)
    sendDialog({
        title   = "⚠️ Key Required",
        message = "Key is invalid or expired.\nReason: " .. msg ..
                  "\n\nGet a key at:\ngrandfuscator.my.id"
    })
    return
end

saveKey(myKey)
LogToConsole("[Key] Valid! Script is running...")

-- PERIODIC RECHECK
local lastCheck = os.time()
runThread(function()
    while true do
        Sleep(60000)
        if os.time() - lastCheck >= KEY_RECHECK then
            lastCheck = os.time()
            local valid, vmsg = validateKey(myKey)
            if not valid then
                LogToConsole("[Key] EXPIRED: " .. vmsg)
                sendDialog({
                    title   = "⛔ Key Expired",
                    message = "Your key is no longer valid.\n" .. vmsg ..
                              "\n\nPlease get a new key."
                })
                stopmyscript()  -- REPLACE with your stop function
                return
            end
        end
    end
end)

-- place your main scripts
```

---

📡 API Reference

I only recommend using my API, if you can create your own API, it doesn't matter if you don't use this API.

Endpoint

```
GET https://www.grandfuscator.my.id/api/grandfuscator-keyvalidation
```

Parameters

Parameter Required Description
name Yes Your panel username (case-sensitive)
checkkey Yes The key to validate

Example Request

```
GET /api/grandfuscator-keyvalidation?name=yourusernameaccount&checkkey=GRAND-KEY-ABCD1234
```

Responses

Valid Key:

```json
{
  "valid": true,
  "message": "Key is valid",
  "expiresAt": 1780000000
}
```

Invalid/Expired Key:

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

---

🔑 Key Format

Keys follow this format:

```
GRAND-KEY-XXXXXXXX
```

Where XXXXXXXX is a random 8-character alphanumeric string (uppercase).

Examples:

```
GRAND-KEY-A3F9K2PQ
GRAND-KEY-ZX17BCDE
GRAND-KEY-90MNRTYV
```

---

❓ FAQ

Q: Can users run the script without a key?
A: No. The script stops at return before reaching your main logic if the key is invalid.

Q: Is the key saved automatically?
A: Yes, the key is saved on the user's device. They only need to input it once unless it expires.

Q: How do users get a new key?
A: Direct them to your public page. Get the URL from the panel → My Key Manager → Public Page.

Q: What should I replace stopmyscript() with?
A: Replace it with your script's actual stop function, for example:

· Stopscript()
· os.exit()

---

Grandfuscator — Lua Key Validation System Documentation
