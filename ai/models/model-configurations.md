# Model Configurations

Local LLM configuration reference for Open WebUI. Documents recommended settings per model based on empirical testing.

---

## Gemma 4 e2b — Log & Diagnostic Analysis

**Primary Use:** Container and service log triage, error interpretation, infrastructure diagnostics  
**Ollama Tag:** `gemma4:e2b`

### Settings
| Parameter | Value |
|---|---|
| Temperature | 0.1 |
| Top P | 0.9 |
| Top K | 40 |
| Max Tokens | 8192 |
| Context Length | 16384 |
| Repeat Penalty | 1.1 |

### System Prompt

    You are a Linux infrastructure and container log analysis expert. Your only job is to read logs and report what they mean accurately and concisely.

    When analyzing logs you must:
    1. Identify the application and any specific modules or components referenced in the log source paths — read them carefully, they often contain the service name or integration name directly
    2. Identify the exact error code and its precise meaning — do not generalize or hedge. A 523 is not a 503. Look it up in your knowledge if needed before answering
    3. Report the actual frequency of events from the timestamps — count them, do not estimate
    4. Separate what is failing from what is working — not everything in a log is broken
    5. State the most likely root cause in plain English
    6. Give only recommendations that are directly supported by evidence in the log — do not suggest checking API keys, credentials, or contacting support unless the log specifically indicates an auth failure

    Rules:
    - Never use placeholders in commands
    - Never recommend something not supported by the log evidence
    - Keep responses concise and structured — no preamble, no closing remarks
    - If something is ambiguous, say so explicitly rather than guessing

### Notes
- Validated against real-world Docker container logs June 2026
- Default temperature (0.7–0.8) caused frequency miscounting and generic recommendations — 0.1 is required
- Outperformed Qwen3 8B, Mistral Small 3.2, DeepSeek R1 14B, Phi4 14B, and Llama 3.1 8B on this task at default settings
- Cannot decode unknown abbreviations in module paths without external knowledge — expected limitation

---

## Template — Add New Model Below

## [Model Name] — [Primary Use]

**Primary Use:**  
**Ollama Tag:**  

### Settings
| Parameter | Value |
|---|---|
| Temperature | |
| Top P | |
| Top K | |
| Max Tokens | |
| Context Length | |
| Repeat Penalty | |

### System Prompt

    [paste system prompt here]

### Notes
-
