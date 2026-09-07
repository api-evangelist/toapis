---
name: Apis
description: Use when building integrations with AI models (GPT-5, Claude, Gemini), generating images, creating videos, or migrating from OpenAI. Reach for this skill when you need to call text/chat APIs, submit async image/video generation tasks, manage API keys, check balances, or configure webhooks for task completion notifications.
metadata:
    mintlify-proj: apis
    version: "1.0"
---

# Apis (ToAPIs) Skill

## Product Summary

Apis (ToAPIs) is a unified OpenAI-compatible API gateway providing access to leading language models (GPT-5, Claude Sonnet, Gemini 2.0) plus async image and video generation APIs. Agents use it to call chat completions, generate images/videos, manage async tasks, and monitor account balance. The primary endpoint is `https://toapis.com/v1` (or `https://toapis.cn` for mainland China). Key files: API keys stored in console, task IDs returned from generation endpoints. CLI: use cURL or SDKs (Python openai, Node.js openai, Java). See [https://docs.toapis.com](https://docs.toapis.com) for full documentation.

## When to Use

- **Chat/text generation**: User asks to call GPT-5, Claude, Gemini, or other text models; needs streaming, vision input, or multi-turn conversation
- **Image generation**: User requests text-to-image or image-to-image generation; needs async task tracking
- **Video generation**: User wants to create videos from text or images; requires polling or webhook monitoring
- **OpenAI migration**: User has OpenAI code and wants to switch providers; only base URL and API key change needed
- **Task management**: User needs to check status of async image/video jobs, configure webhooks, or handle task completion
- **Account monitoring**: User wants to check remaining balance, token usage, or create new API keys
- **File uploads**: User has local images to use in generation APIs; must upload first before using URLs

## Quick Reference

### API Endpoints

| Task | Endpoint | Method |
|------|----------|--------|
| Chat completion | `POST /v1/chat/completions` | OpenAI-compatible |
| List models | `GET /v1/models` | Query available models |
| Image generation | `POST /v1/images/generations` | Async, returns task_id |
| Get image status | `GET /v1/images/generations/{task_id}` | Poll for results |
| Video generation | `POST /v1/videos/generations` | Async, returns task_id |
| Get video status | `GET /v1/videos/generations/{task_id}` | Poll for results |
| Upload image | `POST /v1/uploads/images` | Multipart form, returns URL |
| Check balance | `GET /v1/balance` | Query token usage |
| Create token | `POST /v1/account/tokens` | Generate new API key |

### Authentication

All requests require Bearer token in Authorization header:
```
Authorization: Bearer sk-xxxxxxxxxxxxxxxx
```

API keys start with `sk-` prefix. Get from console at https://toapis.com/console/token.

### Model Selection

| Type | Starter Model | Use Case |
|------|---------------|----------|
| Text | `gpt-5.6-terra` | Balanced performance/cost for chat and agents |
| Image | `gpt-image-2` | Text-to-image and reference-image generation |
| Video | `sora-2-vvip` | Text-to-video, image-to-video, character references |

Use `/v1/models` endpoint to list all available models for current API key.

### Task Status Values

| Status | Meaning | Final? | Action |
|--------|---------|--------|--------|
| `queued` | Waiting in queue | No | Wait 5-10 seconds, poll again |
| `in_progress` | Processing | No | Wait 5-10 seconds (images) or 10-15 seconds (videos), poll again |
| `completed` | Success | Yes | Extract result from `result.data[0].url` |
| `failed` | Error | Yes | Check `error.message` for details |

### Polling Strategy

- **Images**: Initial wait 5s, poll every 5-10s with jitter, max 120s
- **Videos**: Initial wait 5s, poll every 10s with jitter, max 600s (10 min)
- **Rate limits**: Respect `Retry-After` header on 429; use exponential backoff
- **Preferred**: Use webhooks instead of polling; configure in console

## Decision Guidance

### When to Use Chat Completions vs. Responses API

| Scenario | Use Chat Completions | Use Responses API |
|----------|---------------------|-------------------|
| Simple chat, streaming | ✓ | |
| Multi-turn conversation | ✓ | ✓ |
| Function calling, tool use | | ✓ |
| Server-side context management | | ✓ |
| Existing OpenAI SDK code | ✓ | |
| GPT-5 Pro, Gemini 3.1 Pro | | ✓ (Responses Only) |

### When to Upload Images vs. Pass URLs

| Scenario | Upload First | Use Direct URL |
|----------|--------------|-----------------|
| Local file on disk | ✓ | |
| Public image URL | | ✓ |
| Base64 data | ✓ (convert to file) | |
| Reuse same image multiple times | ✓ (upload once) | |
| Large images (>10MB) | ✗ (exceeds limit) | |

### When to Use Webhooks vs. Polling

| Scenario | Use Webhooks | Use Polling |
|----------|--------------|-------------|
| Real-time completion notification | ✓ | |
| Fallback when webhook fails | | ✓ |
| Batch job monitoring | ✓ | |
| Simple one-off request | | ✓ |
| Rate-limited account | ✓ | |

## Workflow

### 1. Chat Completion (Synchronous)

1. Get API key from console; verify it starts with `sk-`
2. Choose model from `/v1/models` or use `gpt-5.6-terra` as default
3. Build messages array with `role` (system/user/assistant) and `content`
4. POST to `/v1/chat/completions` with Authorization header
5. Parse response: extract `choices[0].message.content` for text
6. Check `usage` field for token counts

### 2. Image Generation (Asynchronous)

1. If using local image: upload via `POST /v1/uploads/images`, get URL from response
2. POST to `/v1/images/generations` with model, prompt, and image_urls (if reference)
3. Extract `id` from response (task ID)
4. Poll `GET /v1/images/generations/{task_id}` every 5-10 seconds
5. When status is `completed`, extract URL from `result.data[0].url`
6. Download image within 24 hours (URLs expire)

### 3. Video Generation (Asynchronous)

1. If using reference image: upload via `POST /v1/uploads/images` first
2. POST to `/v1/videos/generations` with model, prompt, duration, aspect_ratio
3. Extract `id` from response (task ID)
4. Poll `GET /v1/videos/generations/{task_id}` every 10 seconds (videos take longer)
5. When status is `completed`, extract URL from `result.data[0].url`
6. Download video within 24 hours (URLs expire)

### 4. Configure Webhooks (Optional)

1. Go to console, edit Token settings
2. Set HTTPS callback URL (e.g., `https://yourapp.com/webhooks/toapis`)
3. Generate signing secret (shown once, save it)
4. Enable Task Webhooks, save
5. In your webhook handler: verify signature using HMAC-SHA256, deduplicate by event ID
6. Extract task result from webhook payload instead of polling

## Common Gotchas

- **API key format**: Must start with `sk-`. If it doesn't, you have the wrong key or it's corrupted.
- **Base URL for China**: Use `https://toapis.cn` instead of `https://toapis.com` for mainland China users; affects all endpoints.
- **Base64 images deprecated**: Do not pass base64 data directly in generation APIs. Upload first, use returned URL.
- **Task URLs expire**: Generated images/videos are valid for 24 hours only. Download and store immediately.
- **Async tasks are truly async**: Chat returns instantly; images/videos return task_id immediately. Must poll or use webhooks.
- **Rate limits are per-user, not per-token**: All tokens owned by one user share the same rate limit bucket.
- **Polling too fast**: Hitting 429 rate limit? Respect `Retry-After` header and add exponential backoff; prefer webhooks.
- **Vision input requires specific models**: Not all models support images. Check model list or docs for vision capability.
- **Streaming incompatible with async**: Chat completions support streaming; image/video generation does not (always async).
- **Missing Authorization header**: All requests fail with 401 if header is missing or malformed. Format: `Authorization: Bearer sk-xxxxx`.
- **Content policy violations**: Some prompts may be rejected with 422. Check error message for details.
- **Insufficient balance**: Requests fail with 402 if token has no remaining balance. Check `/v1/balance` endpoint.

## Verification Checklist

Before submitting work with Apis:

- [ ] API key is valid and starts with `sk-`
- [ ] Using correct base URL (`https://toapis.com/v1` or `https://toapis.cn/v1` for China)
- [ ] Authorization header is present and formatted correctly: `Authorization: Bearer sk-xxxxx`
- [ ] Model name is valid (check `/v1/models` or docs)
- [ ] For chat: messages array has required `role` and `content` fields
- [ ] For images/videos: prompt is provided and not empty
- [ ] For image/video with reference: image uploaded first via `/v1/uploads/images`, URL used in request
- [ ] For async tasks: polling interval is at least 5-10 seconds with jitter, not hammering API
- [ ] For webhooks: signature verified using HMAC-SHA256, event ID deduplicated
- [ ] Token balance checked if requests are failing (use `/v1/balance`)
- [ ] Generated URLs downloaded within 24-hour expiration window
- [ ] Error responses parsed correctly (check `error.code` and `error.message`)

## Resources

- **Full page navigation**: [https://docs.toapis.com/llms.txt](https://docs.toapis.com/llms.txt)
- **Quick Start Guide**: [https://docs.toapis.com/docs/en/quickstart](https://docs.toapis.com/docs/en/quickstart)
- **Chat Completions API**: [https://docs.toapis.com/docs/en/api-reference/chat/chat](https://docs.toapis.com/docs/en/api-reference/chat/chat)
- **Supported Models**: [https://docs.toapis.com/docs/en/api-reference/chat/models](https://docs.toapis.com/docs/en/api-reference/chat/models)

---

> For additional documentation and navigation, see: https://docs.toapis.com/llms.txt