# Cognito Demo

A premium dark-theme client demo featuring AI-powered chatbots, WebGL particle effects, and conversational UI — built as a white-label template for web development clients.

**Live on Cloudflare Workers** | Astro 6 SSR | TypeScript

---

## What This Is

A fully interactive demo site designed to showcase what a modern AI-integrated web presence looks like. Every page has something working — trained chatbots, real-time WebGL, animated scroll interactions, and a conversational contact form that replaces traditional web forms entirely.

Built using an **Anthropic-powered development workflow** with customised Claude instruction files (CLAUDE.md) that enforce architecture patterns, error handling, and deployment safety across the entire build process.

---

## AI Chatbots

The core feature. Three distinct chatbot implementations, all running **Qwen 3 (30B parameters)** on Cloudflare Workers AI with edge inference:

- **AI Concierge** — Hero section greeter with typewriter effect, option buttons, and contextual follow-ups
- **Full Chat Page** — Dedicated conversational interface with session memory (persists across page reloads), typing indicators, and 15-second timeout handling
- **AI Contact Wizard** — Replaces traditional contact forms with a guided conversational brief builder. A split-panel UI where the left side is a chat and the right side is a live-updating project brief

### Chatbot Architecture

| Layer | Detail |
|-------|--------|
| **Smart FAQ Matching** | Common questions get instant answers with zero AI tokens — pricing, services, timelines answered in milliseconds |
| **Adversarial Hardening** | Prompt injection, persona hijacking, jailbreak attempts — tested and sealed. Five explicit hard boundaries in the system prompt |
| **Rate Limiting** | KV-backed per-IP throttling (20 req/min) with graceful user-facing messaging |
| **Lead Capture** | Intent detection triggers automatic conversation storage in KV for follow-up |
| **Think Token Stripping** | Qwen 3's internal `<think>` reasoning tokens are stripped before delivery — users see clean responses only |
| **Fallback Chain** | FAQ match -> Workers AI -> scripted demo responses. Never fails silently |

---

## WebGL & Visual Effects

Full-screen **Three.js particle system** (2000 particles on desktop, 800 on mobile) with custom GLSL shaders:

- **Mouse gravity** — Particles are attracted toward the cursor with spring physics
- **Click shockwaves** — Clicking anywhere sends a radial force wave through nearby particles
- **Scroll-reactive camera** — Camera depth and roll respond to scroll velocity and position
- **Additive blending** — Warm gold particles with proximity-based colour shifting
- **Spring physics engine** — Custom spring system (snappy/bouncy/heavy presets) drives cursor, camera, particles, and tilt effects

Additional visual layers:
- **Custom cursor** with spring-physics dot + ring + sparkle trail
- **3D tilt cards** — Mouse-follow perspective transforms with dynamic shadow casting
- **GSAP ScrollTrigger animations** — Reveal, parallax, stagger, scale, and letter-by-letter effects
- **Lenis smooth scrolling** integrated with GSAP ticker
- **Page transitions** — Directional wipe overlays between routes
- **Animated loader** with brand text and progress bar
- **Ember particles** — CSS-driven floating particles on section headers
- **Rift panel effects** — Glowing corners, scanlines, and ember accents on chat containers

---

## Pages

| Route | What It Does |
|-------|-------------|
| `/` | Hero with AI concierge, services cards, stats with count-up animation, inline AI demo, smart FAQ, chatbot feature showcase (3D tilt card), CTA |
| `/about` | Mission, process, tech stack tags, highlight box |
| `/chat` | Full-page chatbot with session persistence, rift panel effects |
| `/contact` | AI-powered conversational brief wizard with live-updating project summary |

---

## Tech Stack

| Technology | Role |
|-----------|------|
| **Astro 6** | SSR framework with Cloudflare adapter |
| **Cloudflare Workers AI** | Qwen 3 30B inference at the edge |
| **Cloudflare KV** | Session storage, rate limiting, lead capture |
| **Three.js** | WebGL particle system with custom GLSL shaders |
| **GSAP + ScrollTrigger** | Scroll-driven animations and page transitions |
| **Lenis** | Smooth scrolling with GSAP ticker integration |
| **TypeScript** | End-to-end type safety |

---

## Running Locally

```bash
npm install
npm run dev        # Starts on localhost:4321
```

AI chat features require Cloudflare Workers AI and KV bindings. Locally, the chatbot falls back to scripted demo responses automatically.

```bash
npm run build      # Production build
```

---

## Architecture Decisions

- **No frameworks for interactivity** — All chat UIs, the contact wizard, and effects are vanilla TypeScript. Zero client-side framework overhead.
- **FAQ-first, AI-second** — Common questions are pattern-matched and answered instantly. The LLM only runs when needed, keeping costs near zero for typical traffic.
- **Graceful degradation** — No Three.js? No particles. No GSAP? No animations. No AI binding? Scripted responses. Nothing breaks.
- **White-label ready** — Brand name, nav links, colours, and content are all configurable from a single layout file. System prompts and FAQ data are isolated per-client.

---

## Development Workflow

Built with **Claude Code** using a structured Anthropic workflow:

- Custom `CLAUDE.md` instruction files enforce architecture patterns, error handling protocols, and deployment safety
- Two-attempt debugging protocol with escalation
- Skill-based automation for deployment, diagnostics, and verification
- Memory system for cross-session context persistence

---

## License

This is a demo/portfolio piece. Not licensed for redistribution.
