# Gravitee

API Management platform. Mostly complete. Read the simple version first if rusty.

## Status: ✅ Largely done — reference for review

## Files

| File | What it covers |
|---|---|
| [simple.md](simple.md) | "Friend explaining over coffee" — 5 core concepts, airport-security analogy, mental model. **Start here.** |
| [deep-dive.md](deep-dive.md) | Detailed reference: architecture, policy chain, AM, comparison to Kong/Apigee, K8s operator |

## The honest "have you used it?" answer

In `simple.md` and `deep-dive.md`. Practice it out loud — it's the most important sentence for any APIM-related interview where you don't have direct experience.

## When to review

- **Before any interview that mentions API gateway / APIM**: `simple.md` is enough for a 30-min refresh.
- **For role specifically asking about Gravitee/Kong/Apigee**: re-read `deep-dive.md`.
- **For positioning your Istio experience**: the "category framing" section in either file.

## Key gaps if interview probes deep

If the next interviewer probes specific Gravitee operational experience, fall back to the honest answer. Don't bluff:
- Custom policy development (Groovy/JS)
- Gravitee AM federation with external IdPs
- Cockpit multi-environment management
- Production Redis HA + rate-limit counter semantics under failover

These are all "I'd learn on the job; here's how I'd approach it" answers, not bluffs.
