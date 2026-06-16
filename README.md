# DLS AI Chatbot

An intelligent, context-aware AI assistant built for **Digital Logics Studio (Boolforge)**. Powered by the [Groq API](https://console.groq.com/), it delivers fast and highly personalized responses on digital logic, Boolean algebra, number systems, sequential circuits, and all topics covered in the DLS curriculum.

---

## Overview

The DLS AI Chatbot is a dedicated microservice that plugs into the existing Boolforge + DigitalLogicsStudio-Backend ecosystem. It exposes a simple REST endpoint that the frontend calls whenever a user sends a message. On the backend, it builds a rich system prompt from the user's learning context (topics visited, tools used, current topic, logged-in profile) and fires a completion request to Groq, returning a sharp, focused answer in milliseconds.

```
DigitalLogicsStudio-AI/
├── src/
│   ├── config/
│   │   └── groq.js              # Groq client initialization
│   ├── controllers/
│   │   └── chatController.js    # Request handler & response formatter
│   ├── middleware/
│   │   ├── auth.js              # Reuses JWT verification from main backend
│   │   └── rateLimit.js         # Per-user message throttling
│   ├── prompts/
│   │   └── systemPrompt.js      # Personalized system prompt builder
│   ├── routes/
│   │   └── chat.js              # POST /api/ai/chat
│   └── app.js
├── .env.example
├── server.js
├── package.json
└── README.md
```

---

## Features

- **Personalized responses** — The system prompt is dynamically assembled from the user's JWT profile, their current topic, recently visited pages, and tools they have interacted with, so every answer is relevant to exactly where they are in the curriculum.
- **Groq-powered speed** — Uses `llama3-70b-8192` (or `mixtral-8x7b-32768`) via Groq's inference API for near-instant completions, even for complex logic explanations.
- **Curriculum-aware context** — The chatbot knows the full DLS topic tree (Boolean Algebra, Number Systems, Arithmetic Circuits, Memory, Sequential Circuits) and steers answers to match the learner's current module.
- **Streaming support** — Optional SSE streaming endpoint for character-by-character output, giving users the feel of a live expert typing back to them.
- **Rate limiting** — Per-user token bucket prevents API quota abuse without interrupting legitimate learning sessions.
- **JWT-gated** — Reuses the same HTTP-only cookie and JWT secret from the main backend; no second login needed.
- **Graceful fallback** — If Groq is unavailable, returns a structured error with a helpful offline suggestion rather than a raw 500.

---

## Tech Stack

| Layer | Choice |
|---|---|
| Runtime | Node.js 20 |
| Framework | Express |
| AI Provider | Groq Cloud API |
| Model | `llama3-70b-8192` (default) |
| Auth | JWT (shared secret with main backend) |
| Rate Limiting | `express-rate-limit` |
| HTTP Client | `groq-sdk` (official) |
| Streaming | Server-Sent Events (SSE) |

---

## Environment

Create a `.env` file from `.env.example`:

```env
PORT=5100
NODE_ENV=development

# Groq
GROQ_API_KEY=your-groq-api-key-here
GROQ_MODEL=llama3-70b-8192
GROQ_MAX_TOKENS=1024
GROQ_TEMPERATURE=0.5

# Shared with main backend — must match exactly
JWT_SECRET=replace-with-the-same-secret-as-main-backend

# CORS — point at the Boolforge frontend
CLIENT_URL=http://localhost:3000

# Rate limiting (requests per user per minute)
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=20
```

> **Note:** `GROQ_TEMPERATURE=0.5` is intentionally moderate — low enough for factually grounded logic explanations, high enough to keep prose natural. Raise it toward `0.8` for more conversational tone, lower toward `0.2` for stricter technical answers.

---

## Installation

```bash
# From the monorepo root (sibling to DigitalLogicsStudio-Backend)
cd DigitalLogicsStudio-AI

npm install
```

---

## Running

Development (with auto-restart):

```bash
npm run dev
```

Production:

```bash
npm start
```

The service listens on `PORT` (default `5100`) and is completely independent of the main backend process. Run both simultaneously.

---

## API Reference

### `POST /api/ai/chat`

Send a user message and receive a personalized AI response.

**Auth:** Required — valid JWT in `Authorization: Bearer <token>` header or the `token` HTTP-only cookie (same as the main backend).

**Request body:**

```json
{
  "message": "How does a JK flip-flop differ from a SR flip-flop?",
  "context": {
    "currentTopic": "sequential-circuits",
    "recentTopics": ["boolean-algebra", "arithmetic-circuits"],
    "toolsUsed": ["circuit-forge"],
    "difficulty": "intermediate"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `message` | string | Yes | The user's question or message |
| `context.currentTopic` | string | No | Slug of the page the user is currently on |
| `context.recentTopics` | string[] | No | Ordered list of recently visited topic slugs |
| `context.toolsUsed` | string[] | No | Tools the user has interacted with this session |
| `context.difficulty` | string | No | `beginner`, `intermediate`, or `advanced` |

**Response `200`:**

```json
{
  "reply": "Great question! A SR flip-flop has an undefined (invalid) state when both S and R are HIGH simultaneously...",
  "model": "llama3-70b-8192",
  "tokensUsed": 312
}
```

**Response `429`:**

```json
{
  "error": "Too many requests. Please wait a moment before asking again."
}
```

---

### `POST /api/ai/chat/stream` *(optional)*

Same request body as above. Returns a `text/event-stream` (SSE) response that streams the reply token by token.

**Frontend usage:**

```js
const source = new EventSource('/api/ai/chat/stream', { withCredentials: true });
source.onmessage = (e) => appendToken(e.data);
source.onerror = () => source.close();
```

---

## Personalization System

The core of the chatbot's quality is in `src/prompts/systemPrompt.js`. It assembles a system prompt at request time that includes:

```
You are DLS Mentor, an expert teaching assistant for Digital Logics Studio —
an interactive platform for learning digital logic and Boolean algebra.

Student profile:
- Name: {user.name}
- Current topic: {context.currentTopic} (Sequential Circuits)
- Recently studied: Boolean Algebra → Arithmetic Circuits
- Tools used this session: Circuit Forge
- Difficulty level: Intermediate

Curriculum scope:
1. Boolean Algebra (gates, expressions, simplification, De Morgan's)
2. Number Systems (binary, octal, hex, BCD, conversions)
3. Arithmetic Circuits (half adder, full adder, subtractor, comparator)
4. Memory (latches, flip-flops, registers, RAM/ROM)
5. Sequential Circuits (FSMs, counters, shift registers)

Persona and tone:
- Speak directly to {user.name} by name when it feels natural.
- Match your depth to an intermediate learner — skip trivial basics,
  do not assume graduate-level prior knowledge.
- Use concrete examples, truth tables, and circuit analogies liberally.
- If the question is outside digital logic, politely redirect back to the curriculum.
- Keep answers concise but complete. Prefer numbered steps for procedures.
```

This means a beginner on the Boolean Algebra page gets a fundamentally different answer than an intermediate student mid-way through Sequential Circuits — from the same question.

---

## Frontend Integration

Add the following to Boolforge's `src/services/` directory:

```js
// src/services/aiService.js
import axios from 'axios';

const AI_URL = process.env.REACT_APP_AI_URL || 'http://localhost:5100/api/ai';

export const sendMessage = (message, context) =>
  axios.post(`${AI_URL}/chat`, { message, context }, { withCredentials: true });
```

Add `REACT_APP_AI_URL=http://localhost:5100/api/ai` to Boolforge's `.env`.

---

## Adding to the Monorepo

The recommended folder layout alongside the existing projects:

```
DigitalLogicsStudio/          ← Boolforge (React frontend)
DigitalLogicsStudio-Backend/  ← Express + MongoDB backend
DigitalLogicsStudio-AI/       ← this service
```

All three run independently. No changes to the main backend are required — the AI service shares only the `JWT_SECRET` value.

---

## Model Selection

Groq supports several models. Swap `GROQ_MODEL` in `.env` to change:

| Model | Best for | Context window |
|---|---|---|
| `llama3-70b-8192` | Deep, accurate explanations (default) | 8 192 tokens |
| `llama3-8b-8192` | Fast lightweight answers | 8 192 tokens |
| `mixtral-8x7b-32768` | Long multi-turn conversations | 32 768 tokens |
| `gemma2-9b-it` | Instruction-tuned, concise answers | 8 192 tokens |

---

## Rate Limits & Quota

Groq free-tier allows **14 400 requests/day** and **500 000 tokens/minute** (as of mid-2025 — check the [Groq limits page](https://console.groq.com/settings/limits) for current values). The built-in per-user rate limiter (`RATE_LIMIT_MAX=20` per minute) ensures a busy classroom does not exhaust the daily quota.

---

## Notes

- `.env` is git-ignored; `.env.example` is tracked.
- `package-lock.json` is tracked for reproducible installs.
- The service is stateless — conversation history is not persisted server-side. If you want multi-turn memory, send the last N message pairs in the request body and include them in the prompt builder.

---

## License

Proprietary License. All rights reserved.
