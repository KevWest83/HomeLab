# LLM Log Analysis Benchmark — June 2026

## Overview

This document records the methodology and results of a benchmarking exercise conducted in June 
2026 to identify the most effective locally-hosted LLM for Linux infrastructure and container 
log analysis tasks. The goal was to find a model that could accurately interpret real-world 
Docker logs, identify error codes precisely, and provide actionable recommendations — with speed 
as a secondary priority.

All models were run locally via Ollama on a self-hosted inference server equipped with an AMD 
Radeon RX 7900 XT (20GB VRAM) using Open WebUI as the interface.

The broader takeaway from this exercise extends beyond model selection. AI assistants were 
consulted throughout this process for recommendations, and in every case the recommendations 
required verification, correction, or outright rejection based on empirical evidence. This is 
not a failure of the tools — it is a demonstration of how they are meant to be used. AI is an 
augmentation tool. It accelerates research, surfaces options, and reduces legwork. It does not 
replace the judgment, critical thinking, and accountability of the engineer using it.

This needs to be stated directly: AI is not good enough on its own without human intervention. 
This benchmark exists and produced a useful outcome because a human drove every step of it. A 
human questioned the initial recommendations. A human designed the test. A human caught the 
configuration errors that caused two models to appear broken when they were not. A human 
interpreted the results. A human made the final decision. The AI assistants involved got 
multiple things wrong with complete confidence and without any indication of uncertainty. 
Without human judgment and intervention at every stage, the wrong model would have been 
selected based on flawed recommendations and inadequate testing.

AI is a tool that makes humans better. It is not a tool that replaces human thought or 
judgment. The most dangerous pattern in working with AI is substituting its output for 
independent thinking. The most effective pattern is treating its output as a starting point 
that a human being must still verify, test, challenge, and decide upon. This benchmark is a 
practical example of that principle in action.

---

## Methodology

### Test Input
A real-world Docker container log was used as the benchmark input, retrieved via the 
`docker logs` command against a running container. The log spanned approximately 48 hours 
and contained a mix of informational entries, recurring warnings, successful operations, 
and a persistent external connectivity failure. The application and container name have 
been generalized for this document.

### Test Prompt
Each model received the same prompt with no additional context or coaching:

> "Can you interpret this log for me?"

No task-specific system prompt was applied during the initial round of testing. Models were 
evaluated on their default behavior with only context length and token settings configured.

### Scoring Criteria
Each model was evaluated against the following criteria:

| Criterion | Description |
|---|---|
| Error code accuracy | Did the model identify the correct HTTP error code and its precise meaning? |
| Hallucination | Did the model invent facts not present in the log? |
| Source identification | Did the model read module path names to identify the failing component? |
| Factual accuracy | Were timestamps, frequencies, and other log details reported correctly? |
| Output quality | Was the response concise, structured, and free of irrelevant recommendations? |

### Initial Settings
All models in the first round were configured with:
- Max Tokens: 8192
- Context Length: 16384
- Temperature: default (not explicitly set)

---

## Models Tested

| Model | Ollama Tag |
|---|---|
| Gemma 4 e4b | `gemma4:e4b` |
| Gemma 4 12B | `gemma4:12b` |
| Gemma 4 e2b | `gemma4:e2b` |
| Mistral Small 3.2 | `mistral-small3.2` |
| DeepSeek R1 14B | `deepseek-r1:14b` |
| Phi4 14B | `phi4:14b` |
| Phi4 Reasoning 14B | `phi4-reasoning:14b` |
| Qwen3 8B | `qwen3:8b` |
| Llama 3.1 8B | `llama3.1:8b` |

---

## A Note on Pre-Test Recommendations

Prior to testing, three AI assistants (Claude, ChatGPT, and Gemini) were consulted for model 
recommendations. All three independently recommended **Qwen3 8B** as the strongest candidate 
for this task, citing its benchmark scores, structured reasoning capability, and speed on AMD 
hardware. Qwen3 8B finished 7th.

Additionally, Claude initially recommended **Llama 3.3 8B** as a strong contender. This model 
does not exist on Ollama — only a 70B variant is available under that tag. The recommendation 
was not verified before being made. This was caught before any testing was conducted.

Both incidents are noted here as a reminder that AI recommendations must be independently 
verified against current, authoritative sources before acting on them. Empirical testing 
supersedes consensus recommendations regardless of how many sources agree.

---

## Results

### Initial Round — Default Temperature

The Gemma 4 12B and e4b models failed to produce meaningful output in the first round. Both 
returned either no response or a single word. Investigation revealed the cause was insufficient 
context length and token budget, not model capability. Settings were adjusted and both models 
were retested.

The Phi4 Reasoning 14B model returned its internal Chain-of-Thought reasoning trace alongside 
the final answer. This is expected behavior for thinking/reasoning model variants — the 
reasoning trace is architectural, not a configuration issue. The model was disqualified from 
the ranked results on output quality grounds but its underlying analysis was noted as accurate.

### Final Standings

| Rank | Model | Error Code | Hallucination | Accuracy | Output Quality | Notes |
|---|---|---|---|---|---|---|
| 1st | Gemma 4 e4b | ✅ 523 | ✅ None | ✅ Strong | ✅ Structured | Best overall |
| 2nd | Gemma 4 12B | ✅ 523 | ✅ None | ✅ Strong | ✅ Clean | Close second |
| 3rd | Gemma 4 e2b | ✅ 523 | ✅ None | ✅ Good | ✅ Clean | Fastest — selected |
| 4th | Mistral Small 3.2 | ⚠️ Hedged | ✅ None | ⚠️ Partial | ❌ Padded | Misread 523 definition |
| 5th | DeepSeek R1 14B | ⚠️ Hedged | ✅ None | ⚠️ Partial | ❌ Generic | Shallow for a reasoning model |
| 6th | Phi4 14B | ❌ Wrong definition | ❌ Wrong dates | ⚠️ Partial | ⚠️ | Hallucinated log dates |
| 7th | Qwen3 8B | ❌ 503 not 523 | ✅ None | ⚠️ Partial | ❌ Verbose | Misread error code despite recommendations |
| DQ | Phi4 Reasoning 14B | ✅ | ✅ | ✅ | ❌ Chain-of-Thought leak | Thinking model — wrong tool for task |
| Last | Llama 3.1 8B | ❌ Wrong | ❌ Invented app | ❌ Weak | ⚠️ | Hallucinated entirely different application |

---

## Key Findings

**Model size is not the determining variable.** Gemma 4 e2b (2B parameters) outperformed or 
matched models two to seven times its size on this task. Parameter count alone is a poor 
predictor of performance on focused, constrained analytical tasks.

**Context length and token budget matter more than expected.** The Gemma 4 12B and e4b models 
appeared to fail initially due to insufficient token and context settings, not capability. 
Correct configuration recovered both models to top-tier performance. Models should not be 
evaluated at default settings for tasks involving large input payloads.

**Default temperature introduces noise on factual tasks.** Temperature was not explicitly set 
in the first round. Phi4 14B hallucinated the log dates despite them appearing on every line. 
Frequency counts were inaccurate across multiple models. Lowering temperature to 0.1 for the 
final e2b configuration resolved both issues.

**Reasoning models are not automatically better.** DeepSeek R1 14B and Phi4 Reasoning 14B are 
deliberate thinking models. Neither outperformed the Gemma models on this task. For constrained 
log triage with a clear correct answer, reasoning overhead adds latency without accuracy benefit.

**AI recommendations require independent verification.** Three separate AI assistants 
recommended Qwen3 8B. It placed 7th. One assistant recommended a model that does not exist in 
the target format. Benchmark results should always take precedence over consensus 
recommendations regardless of source.

---

## Final Configuration — Gemma 4 e2b

Gemma 4 e2b was selected as the primary log analysis model based on the intersection of speed, 
accuracy, and privacy. While Gemma 4 e4b and 12B both delivered marginally stronger accuracy, 
e2b was the fastest model in the shootout by a significant margin. The accuracy difference 
between e2b and the larger Gemma models was a acceptable tradeoff given what was gained: a 
model small enough to return results quickly without competing for VRAM headroom, and one that 
keeps the entire workload local with no justification to escalate to a larger model or external 
API. For infrastructure log triage where speed and privacy matter, e2b delivers the right 
balance.

Final tuned configuration is documented in:

`ai/models/model-configurations.md`

---

## Environment

| Component | Detail |
|---|---|
| Inference server OS | Debian/Ubuntu LTS |
| GPU | AMD Radeon RX 7900 XT (20GB VRAM) |
| Inference engine | Ollama (bare metal) |
| Interface | Open WebUI (Docker) |
| Test date | June 2026 |
