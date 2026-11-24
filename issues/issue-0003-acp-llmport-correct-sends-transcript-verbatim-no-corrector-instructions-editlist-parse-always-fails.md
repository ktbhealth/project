+++
id = "I-003"
title = "ACP LlmPort.correct sends transcript verbatim — no corrector instructions, EditList parse always fails"
status = "open"
created = "2026-11-24"
priority = "high"
labels = ["evals"]
links = ["T-164"]
+++

Found via llm-cleanup-exp (T-164 follow-up): makeAcpSessionLlm promptText sends envelope.input raw as the ACP prompt — the corrector role gets no instructions. cleanupTranscript passes bare ASR text → agent replies conversationally → EditList JSON.parse fails 18/18 on consult-ptbr-real. Fix: adapter wraps corrector/EditList envelopes with the correction contract (or pipeline embeds instructions in input). Blocks real LLM cleanup in eval pipeline.
