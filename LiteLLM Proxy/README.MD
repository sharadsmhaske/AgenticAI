# Anthropic SDK with Google Gemini via LiteLLM

Use the official Anthropic Python SDK to call Google Gemini models through a local LiteLLM Proxy.

The application uses the Anthropic Messages API format, while LiteLLM translates the request and routes it to the Google Gemini API.

---

## Architecture

```
┌──────────────────────┐
│   VS Code / Jupyter  │
│                      │
│   Anthropic SDK      │
└──────────┬───────────┘
           │
           │ Anthropic Messages API
           ▼
┌──────────────────────┐
│    LiteLLM Proxy     │
│    localhost:4000    │
│                      │
│    config.yaml       │
└──────────┬───────────┘
           │
           │ Gemini API
           ▼
┌──────────────────────┐
│    Google Gemini     │
│                      │
│ Google AI Studio Key │
└──────────────────────┘
```

---

## Project Structure

```
project/
│
├── .env
├── .gitignore
├── config.yaml
├── README.md
└── notebook.ipynb
```

---

## Requirements

- Python 3.12+
- VS Code
- VS Code Jupyter extension
- Google AI Studio API key
- Anthropic Python SDK
- LiteLLM

---

## Setup

### 1. Create a Google API Key

Create a Gemini API key through [Google AI Studio](https://aistudio.google.com/).

Store the key in `.env`:

```
GEMINI_API_KEY=your_google_ai_studio_api_key
```

> Do not commit `.env` to Git.

---

### 2. Create .gitignore

Create a `.gitignore` file:

```
.env
__pycache__/
.ipynb_checkpoints/
.venv/
```

---

### 3. Install Dependencies

Open your Jupyter notebook in VS Code and run:

```python
%pip install litellm anthropic python-dotenv
```

> If you need the LiteLLM proxy server extras (e.g. to resolve `No module named websockets`), use `%pip install "litellm[proxy]"` instead.

After the installation finishes, restart the Jupyter kernel.

In VS Code, click the **Restart** button (↻) in the notebook toolbar at the top of the notebook.

Alternatively, open the Command Palette with:

```
Cmd + Shift + P
```

Then select:

```
Jupyter: Restart Kernel
```

After restarting the kernel, continue running the notebook cells from the beginning.

---

### 4. Verify LiteLLM Installation

Run this in a Jupyter notebook cell:

```python
import litellm
import importlib.metadata

print("LiteLLM:", importlib.metadata.version("litellm"))
```

---

### 5. Configure LiteLLM

Create `config.yaml`:

```yaml
model_list:
  - model_name: gemini
    litellm_params:
      model: gemini/gemini-2.5-flash
      api_key: os.environ/GEMINI_API_KEY
```

**Configuration explained:**

| Key | Description |
|-----|-------------|
| `model_name: gemini` | The model alias your Anthropic SDK will use. |
| `model: gemini/gemini-2.5-flash` | Tells LiteLLM to route requests to the Gemini API. |
| `api_key: os.environ/GEMINI_API_KEY` | Reads the Google API key from the `GEMINI_API_KEY` environment variable. |

---

### 6. Load .env

In the Jupyter notebook:

```python
from dotenv import load_dotenv
import os

load_dotenv()

if not os.getenv("GEMINI_API_KEY"):
    raise RuntimeError("GEMINI_API_KEY not found")

print("Gemini API key loaded")
```

> Never print the complete API key.

---

### 7. Start LiteLLM Proxy

LiteLLM Proxy runs separately from the Jupyter notebook.

Open a VS Code terminal.

If the `litellm` command is available:

```bash
litellm --config config.yaml
```

On macOS, if LiteLLM was installed in the user Python directory:

```bash
~/Library/Python/3.12/bin/litellm --config config.yaml
```

Keep this terminal running while using the notebook.

The proxy will be available at: `http://localhost:4000`

---

### 8. Check the Proxy

From your Jupyter notebook:

```python
import requests

response = requests.get("http://localhost:4000/health/liveliness")

print(response.status_code)
print(response.text)
```

A successful response returns HTTP `200`.

---

### 9. Create the Anthropic Client

The notebook uses the official Anthropic SDK:

```python
from anthropic import Anthropic

client = Anthropic(api_key="anything", base_url="http://localhost:4000")
```

> The `api_key` value (here `"anything"`) is a placeholder — it is not sent to Anthropic. The Anthropic SDK connects to the local LiteLLM Proxy instead of directly to Anthropic. The actual Google API key is handled by LiteLLM. Always pass `api_key` as a keyword argument; positional arguments will raise a `TypeError`.

---

### 10. Send Your First Request

Define the model alias (must match `model_name` in `config.yaml`) and create a message:

```python
model = "gemini"

message = client.messages.create(
    model=model,
    max_tokens=100,
    messages=[
        {
            "role": "user",
            "content": "What is quantum computing?"
        }
    ]
)

print(message.content[0].text)
```

**Request flow:**

```
Anthropic SDK
      ↓
localhost:4000
      ↓
LiteLLM
      ↓
gemini/gemini-2.5-flash
      ↓
Google Gemini API
```

---

### 11. Create a Reusable Function

```python
def ask_gemini(prompt, max_tokens=1000):
    response = client.messages.create(
        model="gemini",
        max_tokens=max_tokens,
        messages=[
            {
                "role": "user",
                "content": prompt
            }
        ]
    )
    return response.content[0].text
```

Use it:

```python
answer = ask_gemini(
    "What are the main differences between Python and JavaScript?"
)

print(answer)
```

---

### 12. Streaming

The Anthropic SDK supports streaming:

```python
with client.messages.stream(
    model="gemini",
    max_tokens=1000,
    messages=[
        {
            "role": "user",
            "content": "Write a short story about a robot exploring Mars."
        }
    ]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

---

### 13. Changing Gemini Models

Update the model in `config.yaml`:

```yaml
model_list:
  - model_name: gemini
    litellm_params:
      model: gemini/YOUR_GEMINI_MODEL
      api_key: os.environ/GEMINI_API_KEY
```

Your Python code can continue using `model="gemini"` unchanged. This lets you swap the underlying Gemini model without modifying your application code.

---

## Troubleshooting

### `zsh: command not found: litellm`

Find the executable from your Jupyter notebook:

```python
import shutil

print(shutil.which("litellm"))
```

If it returns a path such as `/Users/yourname/Library/Python/3.12/bin/litellm`, run:

```bash
~/Library/Python/3.12/bin/litellm --config config.yaml
```

You can also add the directory to your zsh PATH permanently:

```bash
echo 'export PATH="$HOME/Library/Python/3.12/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Now `litellm --config config.yaml` should work.

---

### `No module named websockets`

Install the LiteLLM proxy dependencies:

```python
%pip install -U "litellm[proxy]"
```

After installation, restart the Jupyter kernel using the **Restart** (↻) button in the notebook toolbar.

---

### `ConnectionRefusedError`

If you see `ConnectionRefusedError` when connecting to `http://localhost:4000`, the LiteLLM Proxy is not running.

Start it from the VS Code terminal:

```bash
litellm --config config.yaml
```

Then verify:

```python
import requests

response = requests.get("http://localhost:4000/health/liveliness")

print(response.status_code)
print(response.text)
```

---

### `Anthropic.__init__() takes 1 positional argument`

Always pass `api_key` as a keyword argument:

```python
# Correct
client = Anthropic(api_key="anything", base_url="http://localhost:4000")
```

```python
# Incorrect — raises TypeError
client = Anthropic("anything", base_url="http://localhost:4000")
```

---

## Security

- Never commit your API key to Git.
- Your `.env` should contain: `GEMINI_API_KEY=your_secret_key`
- Your `.gitignore` should contain: `.env`
- Do not put your API key directly into your Python code.
- The Anthropic client only needs a dummy value because LiteLLM handles the actual Gemini authentication.

---

## Summary

```
         Jupyter Notebook
               │
         Anthropic SDK
               │
               ▼
      http://localhost:4000
               │
         LiteLLM Proxy
               │
          config.yaml
               │
               ▼
          Gemini API
               │
       GEMINI_API_KEY
               │
               ▼
        Google Gemini
```

The notebook uses the Anthropic API interface, while LiteLLM handles the provider-specific Gemini API communication.
