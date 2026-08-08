---
id: TASK-025
type: task
title: AWS Bedrock LLM adapter
status: todo
priority: low
story: STORY-014
epic: EPIC-004
repo: thinker-agent
estimate: 4
assignee: unassigned
sprint: sprint-002
created: 2026-08-08
updated: 2026-08-08
labels: [ai, llm, bedrock, aws]
---

# TASK-025: AWS Bedrock LLM adapter

## Description

Implement `BedrockAdapter` which wraps `aiobotocore` / `boto3` to call AWS Bedrock `invoke_model` and `invoke_model_with_response_stream` APIs. Primarily used for Claude-on-Bedrock deployments in enterprise tenants.

## Acceptance Criteria

- [ ] `BedrockAdapter` in `adapters/bedrock.py` implements `BaseLLMAdapter`
- [ ] Supports `anthropic.claude-opus-4-5-v1` and `anthropic.claude-3-5-sonnet-20241022-v2:0` model IDs
- [ ] Uses Anthropic's Bedrock message format (same JSON schema as Anthropic API)
- [ ] Reads `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` from environment
- [ ] Streaming via `invoke_model_with_response_stream` with event processing
- [ ] Conformance test suite passes (with mocked boto3 responses)

## Technical Notes

Use `aiobotocore` for async support. Bedrock's Anthropic-family model format is identical to the direct Anthropic API, so the payload construction from `ClaudeAdapter` can be reused. The key difference is the authentication (SigV4 signing done by boto3) and the endpoint URL.
