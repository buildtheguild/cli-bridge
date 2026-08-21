# Custom applications + CLIBridge

**Status: Direct API match**

If your application can make an HTTP request, it can call CLIBridge directly without an OpenAI SDK.

The main endpoint is:

```text
POST /v1/chat/completions
```

---

## Basic curl request

```bash
curl https://your-domain.com/v1/chat/completions \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.5",
    "messages": [
      {
        "role": "system",
        "content": "You are a helpful assistant."
      },
      {
        "role": "user",
        "content": "Explain CLIBridge in one sentence."
      }
    ]
  }'
```

---

## JavaScript / TypeScript with `fetch`

```js
const baseUrl =
  process.env.CLIBRIDGE_BASE_URL ?? "http://localhost:3900/v1";

const response = await fetch(`${baseUrl}/chat/completions`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.CLIBRIDGE_TOKEN}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    model: process.env.CLIBRIDGE_MODEL ?? "gpt-5.5",
    messages: [
      {
        role: "user",
        content: "Hello from my custom application.",
      },
    ],
  }),
});

if (!response.ok) {
  throw new Error(`CLIBridge returned ${response.status}: ${await response.text()}`);
}

const result = await response.json();

console.log(result.choices?.[0]?.message?.content);
```

---

## Model discovery

```bash
curl https://your-domain.com/v1/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

Use one of the returned model IDs.

---

## Streaming with curl

```bash
curl -N https://your-domain.com/v1/chat/completions \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.5",
    "stream": true,
    "messages": [
      {
        "role": "user",
        "content": "Count from one to five."
      }
    ]
  }'
```

The stream uses OpenAI-style Server-Sent Events and ends with:

```text
data: [DONE]
```

---

## Image input

CLIBridge accepts image content parts.

Example:

```json
{
  "model": "gpt-5.5",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "Describe this image."
        },
        {
          "type": "image_url",
          "image_url": {
            "url": "https://example.com/image.jpg"
          }
        }
      ]
    }
  ]
}
```

The selected backend/model must also be capable of handling the image request.

---

## Supported request surface

For maximum compatibility with the current bridge, keep Chat Completions requests to:

```text
model
messages
stream
response_format
```

plus supported image content parts.

Do not assume CLIBridge implements the full OpenAI API.
