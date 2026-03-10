# Authentication Guide

## API Host

**Production URL**: `https://bopenapi.bgwapi.io`

---

## Getting Your API Credentials

Each partner receives a unique **API Key** and **API Secret** from the Bitget Wallet integration team.

- Contact your Bitget Wallet integration representative to apply for credentials
- Default rate limits are defined per endpoint; request higher limits through your representative if needed
- Each API Key is bound to specific server IPs — provide your IP whitelist to your representative before going live

### Testing Credentials

For development and testing, you can use the public test credentials (no IP whitelist required, 2 QPS limit):

```
API Key:    6AE25C9BFEEC4D815097ECD54DDE36B9A1F2B069
API Secret: C2638D162310C10D5DAFC8013871F2868E065040
```

> ⚠️ **Do not use test credentials in production.** They have strict rate limits and are shared by all developers.

---

## Request Authentication

### Required Headers

Every API request must include these headers:

| Header | Description |
|--------|-------------|
| `x-api-key` | Your API Key |
| `x-api-timestamp` | Request timestamp in milliseconds (must be within ±10 minutes of server time) |
| `x-api-signature` | HMAC-SHA256 signature (see below) |

> **Note**: Swap Order endpoints use `Partner-Code` header instead of the HMAC signature headers. Market Data and Token endpoints use the HMAC authentication described here.

---

## Signature Algorithm

### Step 1: Build the Signature Content

Construct a JSON object with the following fields, **sorted by key in lexicographic order**:

| Field | Value |
|-------|-------|
| `apiPath` | Request path without query parameters (e.g., `/bgw-pro/market/v3/coin/getKline`) |
| `body` | Request body as a JSON string (for POST requests) |
| `x-api-key` | Your API Key |
| `x-api-timestamp` | Timestamp string (milliseconds) |
| Any query params | Key-value pairs from the URL query string |

**Example**: For a request to `/swap/api/test?param1=test1&param2=test2` with body `{"data":"test"}`:

```json
{
  "apiPath": "/swap/api/test",
  "body": "{\"data\":\"test\"}",
  "param1": "test1",
  "param2": "test2",
  "x-api-key": "your_api_key",
  "x-api-timestamp": "17200001"
}
```

### Step 2: Generate the Signature

1. JSON-serialize the sorted content object
2. Compute HMAC-SHA256 using your **API Secret** as the key
3. Base64-encode the result

---

## Code Examples

### Python

```python
import hmac
import hashlib
import base64
import json
import time

def signature(path, api_key, api_secret, timestamp, query=None, body=""):
    content = {
        "apiPath": path,
        "body": body,
        "x-api-key": api_key,
        "x-api-timestamp": str(timestamp),
    }
    if query:
        content.update(query)
    
    # Sort keys lexicographically
    sorted_content = json.dumps(dict(sorted(content.items())), separators=(',', ':'))
    
    # HMAC-SHA256 + Base64
    mac = hmac.new(api_secret.encode(), sorted_content.encode(), hashlib.sha256)
    return base64.b64encode(mac.digest()).decode()


# Usage
api_key = "YOUR_API_KEY"
api_secret = "YOUR_API_SECRET"
timestamp = str(int(time.time() * 1000))
path = "/bgw-pro/market/v3/coin/getKline"
body = json.dumps({"chain": "sol", "contract": "...", "period": "1h", "size": 100})

sig = signature(path, api_key, api_secret, timestamp, body=body)

headers = {
    "Content-Type": "application/json",
    "x-api-key": api_key,
    "x-api-timestamp": timestamp,
    "x-api-signature": sig,
}
```

### Node.js

```javascript
const crypto = require("crypto");

function getSignature(apiPath, body, apiKey, apiSecret, timestamp, queryParams = {}) {
  const content = {
    "apiPath": apiPath,
    "body": body,
    "x-api-key": apiKey,
    "x-api-timestamp": timestamp,
  };

  // Add query params
  for (const [key, value] of Object.entries(queryParams)) {
    content[key] = String(value);
  }

  // Sort keys lexicographically
  const sortedKeys = Object.keys(content).sort();
  const sortedContent = Object.fromEntries(sortedKeys.map(key => [key, content[key]]));
  const payload = JSON.stringify(sortedContent);

  return crypto
    .createHmac("sha256", apiSecret)
    .update(payload)
    .digest("base64");
}

// Usage
const apiKey = "YOUR_API_KEY";
const apiSecret = "YOUR_API_SECRET";
const timestamp = Date.now().toString();
const path = "/bgw-pro/market/v3/coin/getKline";
const body = JSON.stringify({ chain: "sol", contract: "...", period: "1h", size: 100 });

const sig = getSignature(path, body, apiKey, apiSecret, timestamp);
```

### Go

```go
func signature(path, apiKey, apiSecret, timestamp string, query map[string]string, body string) string {
    contentMap := make(map[string]string)
    contentMap["x-api-key"] = apiKey
    contentMap["x-api-timestamp"] = timestamp
    contentMap["apiPath"] = path
    contentMap["body"] = body
    for key, value := range query {
        contentMap[key] = value
    }
    content, _ := json.Marshal(contentMap)
    mac := hmac.New(sha256.New, []byte(apiSecret))
    mac.Write(content)
    return base64.StdEncoding.EncodeToString(mac.Sum(nil))
}
```

### Java

```java
public static String signature(String path, String apiKey, String apiSecret, 
                                String timestamp, Map<String, String> query, String body) throws Exception {
    TreeMap<String, String> mapping = new TreeMap<>();
    mapping.put("x-api-key", apiKey);
    mapping.put("x-api-timestamp", timestamp);
    mapping.put("apiPath", path);
    mapping.put("body", body);
    
    if (query != null) {
        mapping.putAll(query);
    }
    
    String content = new Gson().toJson(mapping);
    
    Mac sha256HMAC = Mac.getInstance("HmacSHA256");
    SecretKeySpec secretKey = new SecretKeySpec(apiSecret.getBytes(), "HmacSHA256");
    sha256HMAC.init(secretKey);
    byte[] mac = sha256HMAC.doFinal(content.getBytes());
    
    return Base64.getEncoder().encodeToString(mac);
}
```

---

## FAQ

### Q: Getting 403 Forbidden?

Check in this order:
1. **IP whitelist** — Is your server IP registered with the Bitget Wallet team?
2. **API Key** — Is `x-api-key` correct?
3. **Timestamp** — Is `x-api-timestamp` within ±10 minutes of server time? Use milliseconds.
4. **Signature** — Common signature errors:
   - Keys not sorted lexicographically
   - `body` field not properly JSON-stringified
   - Using seconds instead of milliseconds for timestamp
   - `apiPath` includes query parameters (it should NOT)

### Q: Getting 429 Too Many Requests?

You've hit the rate limit. Options:
1. Reduce request frequency
2. Use batch endpoints (`batchGetBaseInfo`, `batchGetTxInfo`) instead of multiple single calls
3. Contact your representative to apply for higher rate limits

### Q: Can I use the test API Key in production?

**No.** The test key has a 2 QPS rate limit and is shared by all developers. Apply for your own production key through your integration representative.

### Q: How do I get a production API Key?

Contact your Bitget Wallet integration representative. You'll need to provide:
1. Your company/project name
2. Expected request volume (QPS)
3. Server IP addresses for whitelisting

### Q: Do Swap Order endpoints use the same authentication?

Swap Order endpoints (`/bgw-pro/swapx/order/*`) use the `Partner-Code` header instead of HMAC signature. Your partner code is provided separately from your API Key.

```bash
# Market Data — uses HMAC auth
curl -X POST 'https://bopenapi.bgwapi.io/bgw-pro/market/v3/coin/getKline' \
  -H 'x-api-key: YOUR_KEY' \
  -H 'x-api-timestamp: 1710000000000' \
  -H 'x-api-signature: YOUR_SIG' \
  -d '...'

# Swap Order — uses Partner-Code
curl -X POST 'https://bopenapi.bgwapi.io/bgw-pro/swapx/order/getSwapPrice' \
  -H 'Partner-Code: YOUR_PARTNER_CODE' \
  -d '...'
```

### Q: Signature works locally but fails on server?

Most likely a **clock sync issue**. Ensure your server time is accurate (use NTP). The timestamp must be within ±10 minutes of Bitget's server time.
