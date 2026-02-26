# The Web Has Never Been Designed for AI Agents. We're Trying to Fix That.

*By Nick Allain*

---

The web was built for humans. Every design decision — URLs, HTML, navigation, search — assumes a person on the other end. Someone with eyes, patience, and the cognitive ability to extract meaning from a page that's 40% ads, 20% cookie banners, and 40% actual content.

Search engines came along and we adapted. We built sitemaps, structured data, `robots.txt`. We created an entire discipline — SEO — to help crawlers understand our content. It worked because crawlers had a simple, fixed goal: index text, rank pages.

AI agents are different. And the web has no idea what to do with them.

---

## The Problem Nobody's Talking About

Right now, when an AI agent visits your website to complete a task on behalf of a user, here's what happens:

1. It fetches your HTML
2. It strips the navigation, ads, and boilerplate
3. It does its best to infer what your site can actually *do*
4. It takes a guess

That's it. There's no handshake. No negotiation. No way for your site to say: "Here's what I offer. Here's how to ask for it. Here's what I can do that a static page can't."

The agent is flying blind. And your site is a passive data source — not a participant in the interaction.

This matters more than most people realize. AI agents aren't just reading content anymore. They're booking appointments, placing orders, filing requests, answering questions on behalf of users. The gap between "the agent understood your site" and "the agent misunderstood your site" isn't a UX problem. It's a correctness problem.

---

## What We Built

The **Agent Handshake Protocol (AHP)** is an open standard that gives websites a way to speak agent-native. It's not a replacement for your existing site — it's a layer on top of it. A machine-readable manifest that tells any compliant agent exactly what your site offers and how to interact with it.

Discovery is intentionally simple. An agent checks `/.well-known/agent.json`. That's it — one deterministic HTTP request. No scraping, no inference, no guessing.

The manifest describes three interaction modes:

**MODE1 — Static.** Structured content delivery. Think `llms.txt` but with a schema. Compatible with every existing AI tool that ingests text. Zero infrastructure required beyond a static file.

**MODE2 — Conversational.** Your site exposes a RAG-backed endpoint. The agent asks questions in natural language; you return grounded, structured answers from your own knowledge base. The agent stops hallucinating about what you offer because it's asking you directly.

**MODE3 — Agentic.** Your site exposes tools. Real actions — place an order, check inventory, escalate to a human, start an async job. The agent doesn't just read your site. It *works with* your site.

Each mode is backwards compatible. You can ship MODE1 today with a static JSON file and upgrade to MODE2/MODE3 when you're ready.

---

## This Is an Infrastructure Problem

I want to be direct about why we built a protocol rather than a library or a product.

Libraries solve individual problems. Protocols coordinate ecosystems.

Without a shared standard, every AI platform handles agent-to-site interaction differently. OpenAI does it one way. Anthropic another. LangChain another. Site owners have to build N integrations, one per platform. Agents have to learn N different interaction patterns, one per site. Nobody wins.

A protocol means an agent that implements AHP can interact with any AHP-compliant site. A site that implements AHP works with any AHP-compliant agent. One integration, universal reach. That's the bet.

We've also built platform bridges: an MCP endpoint (for Claude and any MCP-compatible agent) and an OpenAPI spec auto-generated from the manifest (for GPT Actions and anything OpenAPI-compatible). The underlying concierge is the same — the platforms just get different access paths.

---

## Where It Stands

The spec is at v0.1. The reference implementation passes 24 of 25 tests (the failing one is an intentional demo-mode deviation — auth enforcement is disabled on the reference server and labeled a known defect).

The test suite is open source. The spec is CC BY 4.0. The reference implementation is MIT.

We're in the process of registering the `agent.json` well-known URI and `ahp-manifest` link relation with IANA — not because we need permission, but because standards need homes.

What we need from the developer community: implementation feedback, edge cases we haven't thought of, and people willing to build AHP support into LangChain, CrewAI, AutoGPT, and the rest of the agent framework ecosystem.

---

**Spec:** [agenthandshake.dev](https://agenthandshake.dev)  
**Reference implementation:** [ref.agenthandshake.dev](https://ref.agenthandshake.dev)  
**GitHub:** [github.com/AHP-Organization](https://github.com/AHP-Organization)

If this is interesting to you, the best thing you can do is read the spec, try the reference implementation, and tell us where it breaks.

---

*Nick Allain is building AHP — an open protocol for agent-native web interactions. Follow progress at agenthandshake.dev.*
