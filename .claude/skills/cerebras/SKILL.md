---
name: cerebras-inference
description: Use this to write code to call an LLM using LiteLLM and OpenRouter with the Cerebras inference provider
---

# Calling an LLM via Cerebras

These instructions allow you write code to call an LLM with Cerebras specified as the inference provider.  
This method uses LiteLLM and OpenRouter.

## Setup

The OPENAI must be set in the .env file and loaded in as an environment variable.  

The uv project must include openAI and pydantic.
`uv add openai pydantic`

## Code snippets

Use code like these examples in order to use Cerebras.

### Imports and constants

```python
from openai import OpenAI
client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
MODEL = "gpt-4o"
```

### Code to call via Cerebras for a text response

```python
response = client.chat.completions.create(model=MODEL, messages=messages)
result = response.choices[0].message.content
```

### Code to call via Cerebras for a Structured Outputs response

```python
response = client.beta.chat.completions.parse(model=MODEL, messages=messages, response_format=MyBaseModelSubclass)
result_as_object = response.choices[0].message.parsed
```