# Agent Handshake Protocol (AHP)

**The open protocol for how AI agents discover and interact with websites.**

---

The web was built for humans. When AI agents visit websites today, they scrape HTML, parse markdown dumps, and guess at structure — the equivalent of handing someone an entire encyclopedia when they asked one question.

AHP defines a better contract. A site publishes a machine-readable manifest. Visiting agents discover it, understand what the site can do, and interact through a structured protocol — asking for exactly what they need and getting exactly that back.

No scraping. No guessing. A handshake.

> AHP is the infrastructure layer for agent-native web presence — the successor to SEO in a world where the agent is the user.

---

## The Three Modes

AHP is designed for progressive adoption. Start with MODE1 in an afternoon. Upgrade when you're ready.

### MODE1 — Static Serve
The agent-readable web. A manifest at `/.well-known/agent.json` points visiting agents to your content. Compatible with existing `llms.txt` implementations — add the manifest and you're done. No server logic required.

### MODE2 — Interactive Knowledge
Visiting agents ask questions. Your site answers from its content. A `POST /agent/converse` endpoint backed by your knowledge base returns precise, sourced answers instead of a document dump. Agents get what they need in a fraction of the tokens.

### MODE3 — Agentic Desk
Your site's agent has tools. It can access databases, call APIs, connect to MCP servers, and escalate to a human — capabilities the visiting agent may not have. A visiting agent delegates tasks; your concierge handles them and delivers results, synchronously or async.

Each mode is backwards compatible with the one before it. A MODE3 site is also a valid MODE1 site.

---

## How Discovery Works

AHP meets agents where they are. Visiting agents find your endpoint through multiple paths:

1. **In-page notice** — an `<section aria-label="AI Agent Notice">` block in the page body. Agents using headless browsers encounter this directly when reading the page.
2. **HTML link tag** — `<link rel="agent-manifest">` in the document `<head>`
3. **Accept header** — respond to `Accept: application/agent+json` with the manifest
4. **Direct fetch** — `GET /.well-known/agent.json`

---

## Quick Look

**The manifest** (`/.well-known/agent.json`):
```json
{
  "ahp": "0.1",
  "name": "My Site",
  "modes": ["MODE1", "MODE2"],
  "endpoints": {
    "converse": "/agent/converse",
    "content": "/llms.txt"
  },
  "capabilities": [
    {
      "name": "content_search",
      "description": "Find posts and pages by topic",
      "mode": "MODE2"
    },
    {
      "name": "contact",
      "description": "How to reach the site owner",
      "mode": "MODE1"
    }
  ],
  "authentication": "none",
  "rate_limits": {
    "unauthenticated": { "requests": "30/minute" }
  },
  "content_signals": {
    "ai_train": false,
    "ai_input": true,
    "search": true
  }
}
```

**A query** (`POST /agent/converse`):
```json
{
  "ahp": "0.1",
  "capability": "content_search",
  "query": "What has the site owner written about AI agents?",
  "context": {
    "requesting_agent": "my-research-bot/1.0"
  }
}
```

**The response**:
```json
{
  "status": "success",
  "response": {
    "answer": "The site owner has written three pieces on AI agents...",
    "sources": [
      {
        "title": "Why Agents Need Better Web Protocols",
        "url": "/blog/agent-protocols",
        "relevance": "direct"
      }
    ]
  },
  "meta": {
    "tokens_used": 187,
    "content_signals": { "ai_train": false, "ai_input": true }
  }
}
```

---

## Implementing AHP

### For site owners

**MODE1 in 5 minutes:**
1. Create `/.well-known/agent.json` with your manifest
2. Add `<link rel="agent-manifest" href="/.well-known/agent.json" type="application/agent+json">` to your HTML `<head>`
3. Add the in-page agent notice to your page body
4. Point `endpoints.content` at your existing `/llms.txt` or a markdown page

If you already have `llms.txt`, you're most of the way there.

**MODE2:** Build a `POST /agent/converse` endpoint. Back it with whatever you have — a vector DB, a structured JSON file, a CMS API. Return answers with source URLs.

**MODE3:** Extend your concierge with tool access. Declare async capabilities. Implement the callback/polling pattern for long-running tasks.

### For agent developers

Implement the discovery flow (Section 3 of the spec). Respect `content_signals`. Honor `Retry-After` on 429 responses. Declare `requesting_agent` in your request context.

---

## The Spec

The full protocol specification is in [`SPEC.md`](./SPEC.md).

It covers:
- Complete manifest and endpoint schemas
- All three modes in detail
- Multi-turn session protocol
- Content signals
- Trust & identity model
- Async / human-escalation flow
- Rate limiting (headers, recommended limits, cost-based throttling)
- Security considerations
- Worked examples

---

## Status

**Draft 0.1.** This is an active working draft, not a final specification. Things will change.

We are looking for:
- Feedback on the protocol design
- Implementers willing to prototype against the spec
- Edge cases we haven't considered

Open an issue. We want the hard questions.

---

## Contributing

See [`CONTRIBUTING.md`](./CONTRIBUTING.md) *(coming soon)*.

The short version: open an issue to propose a change, discuss it there, submit a PR that implements the agreed change. No change to the spec is too small to warrant a discussion first.

---

## License

Specification text: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
Code and schemas: [MIT](./LICENSE)

© 2026 Nick Allain and contributors.
