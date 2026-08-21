# OpenAI SDKs + CLIBridge

**Status: Direct API match for Chat Completions**

The official OpenAI Node.js and Python SDKs allow a custom API Base URL.

Use the SDK's **Chat Completions** API and point it at CLIBridge.

Do not use the OpenAI Responses API with CLIBridge.

---

## Node.js

Install:

```bash
npm install openai
```

Create `example.mjs`:

```js
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.CLIBRIDGE_TOKEN,
  baseURL: process.env.CLIBRIDGE_BASE_URL ?? "http://localhost:3900/v1",
});

const response = await client.chat.completions.create({
  model: process.env.CLIBRIDGE_MODEL ?? "gpt-5.5",
  messages: [
    {
      role: "user",
      content: "Hello from the OpenAI Node.js SDK through CLIBridge.",
    },
  ],
});

console.log(response.choices[0]?.message?.content);
```

Run:

```bash
CLIBRIDGE_TOKEN="YOUR_BRIDGE_TOKEN" \
CLIBRIDGE_BASE_URL="https://your-domain.com/v1" \
CLIBRIDGE_MODEL="gpt-5.5" \
node example.mjs
```

Use a model returned by `/v1/models`.

---

## Node.js streaming

```js
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.CLIBRIDGE_TOKEN,
  baseURL: process.env.CLIBRIDGE_BASE_URL ?? "http://localhost:3900/v1",
});

const stream = await client.chat.completions.create({
  model: process.env.CLIBRIDGE_MODEL ?? "gpt-5.5",
  messages: [
    {
      role: "user",
      content: "Write a short greeting.",
    },
  ],
  stream: true,
});

for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content ?? "");
}
```

---

## Python

Install:

```bash
pip install openai
```

Create `example.py`:

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["CLIBRIDGE_TOKEN"],
    base_url=os.environ.get(
        "CLIBRIDGE_BASE_URL",
        "http://localhost:3900/v1",
    ),
)

response = client.chat.completions.create(
    model=os.environ.get("CLIBRIDGE_MODEL", "gpt-5.5"),
    messages=[
        {
            "role": "user",
            "content": "Hello from the OpenAI Python SDK through CLIBridge.",
        }
    ],
)

print(response.choices[0].message.content)
```

Run:

```bash
CLIBRIDGE_TOKEN="YOUR_BRIDGE_TOKEN" \
CLIBRIDGE_BASE_URL="https://your-domain.com/v1" \
CLIBRIDGE_MODEL="gpt-5.5" \
python example.py
```

---

## Python streaming

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["CLIBRIDGE_TOKEN"],
    base_url=os.environ.get(
        "CLIBRIDGE_BASE_URL",
        "http://localhost:3900/v1",
    ),
)

stream = client.chat.completions.create(
    model=os.environ.get("CLIBRIDGE_MODEL", "gpt-5.5"),
    messages=[
        {
            "role": "user",
            "content": "Write a short greeting.",
        }
    ],
    stream=True,
)

for chunk in stream:
    if chunk.choices:
        print(chunk.choices[0].delta.content or "", end="", flush=True)
```

---

## List models

You can also call the models endpoint through the SDK.

Node.js:

```js
const models = await client.models.list();

for (const model of models.data) {
  console.log(model.id);
}
```

Python:

```python
models = client.models.list()

for model in models.data:
    print(model.id)
```

---

## Important: do not use Responses API

Use:

```text
client.chat.completions.create(...)
```

Do not use:

```text
client.responses.create(...)
```

because CLIBridge does not currently expose `/v1/responses`.

---

## Keep request parameters minimal

For the safest compatibility, start with:

```text
model
messages
stream
response_format
```

Do not add optional OpenAI parameters unless they are documented by the CLIBridge version you are running.
