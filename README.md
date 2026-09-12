# Weather Chatbot (n8n)

A chat agent that answers weather questions using live data from OpenWeatherMap, then sends a heat alert to Slack or email when the temperature crosses a threshold.

Built in n8n. No code to deploy. Import one JSON file, connect the credentials, run it.

## How it works

```
Chat message
     |
  AI Agent  <-- Groq (llama-3.3-70b)
     |      <-- Simple Memory
     |      <-- OpenWeatherMap (tool)
     |
     |  returns JSON: {city, temperature, condition, message}
     |
  Code node  (parses the JSON)
     |
   Switch  on temperature
     |
     +-- hot    --> LLM writes advisory --> Slack
     +-- warm   --> LLM writes advisory --> Gmail
     +-- normal --> Gmail (plain message)
```

Three things make this more than a basic chatbot:

- **The agent must use the tool.** Its system prompt forbids answering weather questions from memory. Every number comes from the API.
- **It returns JSON, not a sentence.** Four named fields. This is what lets the Switch node compare the real temperature instead of guessing from text.
- **It acts on the result.** Above a threshold it writes an advisory and sends it somewhere, instead of just replying in chat.

## Stack

| Part | Used |
|---|---|
| Workflow engine | n8n |
| Model | Groq, llama-3.3-70b-versatile |
| Weather data | OpenWeatherMap |
| Memory | n8n Simple Memory |
| Alerts | Slack, Gmail |

## Setup

You need API keys for Groq, OpenWeatherMap, Slack (OAuth2), and Gmail (OAuth2).

1. Add all four as credentials in n8n under Settings, Credentials.
2. Import `workflows/weather_chatbot.json`.
3. Open every node marked `REPLACE_WITH_YOUR_...` and pick your own credential, Slack channel, and email address. There are three Groq nodes, and all three need a credential.
4. Apply the two fixes below.
5. Open the chat panel and test.

A new OpenWeatherMap key can take up to an hour to start working.

## Test cases

| Input | Expected |
|---|---|
| `What is the weather?` | Asks which city |
| `Lahore` | Full report with all fields |
| `Is it hot today?` | Reuses Lahore from memory |
| `Tomorrow's weather?` | Refuses, says current weather only |
| `Lahore ka mausam?` | Replies in Roman Urdu |

## Known issues

**1. The Gmail alert sends an empty message.** The `Send a message1` node has a hardcoded subject and body, so it throws away the advisory the LLM just wrote. Set the message to `={{ $json.text }}`.

**2. One Switch branch never runs.** The rules are `>35`, `<35`, `<30`. n8n uses the first match, so anything below 30 is caught by `<35` first and the third branch is dead. Exactly 35 matches nothing at all and disappears silently. Change the rules to `>=35`, then `>=30 and <35`, then `<30`.

**3. Three different thresholds.** The system prompt uses 35 and 10, the Switch uses 35 and 30, and the advisory prompt uses 30. Pick one set and use it everywhere.

**4. A missing city breaks the Switch.** When no city is given, `temperature` is null and the Switch comparison fails. Add an IF node before it to handle the null case.

**5. No rate limit.** Asking five times sends five alerts.

## Note on the JSON

Credential IDs, webhook IDs, the Slack channel ID, and the email address have been replaced with placeholders. n8n exports do not contain API keys, but they do contain IDs tied to one instance, and those should not be committed.

## License

MIT
