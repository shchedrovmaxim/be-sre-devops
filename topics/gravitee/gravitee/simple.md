# Gravitee — Simple Version (the friend-explaining-over-coffee edition)

> Read this first. If you only have time for one document, this is it.

---

## The big idea in 2 sentences

**Gravitee is the place where you turn your internal APIs into a product that other people (mobile apps, partners, internal teams) can sign up for and use.** Think of it like an app store, but for APIs — you publish your API, people register their app, get a key, and call your API within the limits of whichever plan they signed up for.

That's the whole thing.

---

## The mental model — your API is a product, not a URL

When you built apps before, an API was just a URL. Anyone with network access could hit it. Maybe you put a JWT check in middleware. Done.

**API Management flips this.** Your API is now a *product*. To use it, someone has to:

1. **Register** themselves (or their app) — like signing up for Spotify.
2. **Choose a plan** — Free? Gold? Enterprise? Each has different limits and prices.
3. **Get credentials** — an API key, or OAuth credentials. Like a Spotify login.
4. **Call the API** within the plan's rules — exceed the rate limit, you get blocked.
5. **See their own dashboards** — how many calls they made, error rates, etc.

You, the API owner, see all of this in a control panel. You can see "Partner X used 4,200 calls today, mostly successful" or "Plan Free has 50,000 active consumers."

This is the **product layer** that Istio doesn't have. Istio just routes traffic; Gravitee treats traffic as a thing sold to customers.

---

## The 5 things you need to know (with simple analogies)

Imagine you run a restaurant.

### 1. API = your restaurant
The thing customers come to consume. It has a name, an address (URL path), a menu (what endpoints exist), and a back kitchen (your backend service).

Example: "Wallet API" — endpoint `/v2/wallet/*`, backend `wallet-svc.payments.svc.cluster.local`.

### 2. Plan = the menu category
Like "Lunch menu" vs "Dinner menu" vs "Tasting menu" — same restaurant, different rules.

A Plan defines:
- **How you prove who you are** (show your ID = API key, JWT, OAuth)
- **How much you can order** (rate limit: 100 req/min, 1M req/month)
- **What it costs** (free, paid)

Same API can offer multiple Plans at the same time:
- **Public Beta** — anyone can use, but only 60 req/min
- **Gold Partner** — needs OAuth, 10,000 req/min
- **Internal** — needs a specific JWT, no limit

### 3. Application (= Consumer) = your customer
The person or app that wants to use your API. Could be:
- A consumer-facing mobile app
- A partner integration
- An internal microservice

In Gravitee, when someone wants to call your API, they first **register their Application**. They get an account.

### 4. Subscription = the actual relationship
When an Application picks a Plan and signs up, that creates a Subscription. Think of it as the receipt that says "Mobile App X (Application) is subscribed to Wallet API (API) on the Gold Plan (Plan)."

The Subscription is where Gravitee stores their API key. When their request comes in with that key, Gravitee knows: "This is the Gold Plan subscription, apply Gold's rate limit."

### 5. Policy = a step that happens to every request
A policy is just **a function that runs when a request comes in**. Same idea as Express middleware:

```js
app.use(checkApiKey)
app.use(rateLimit)
app.use(transformHeaders)
```

Each one is a "policy." Gravitee has a built-in library of them: `api-key`, `jwt`, `rate-limit`, `quota`, `cache`, `transform-headers`, etc. You arrange them in order.

---

## How a request flows through Gravitee (the airport security analogy)

Imagine your request is a passenger at an airport. Going through Gravitee is like going through security and the gate:

```
Passenger walks in
       ↓
1. Check your ticket            ← api-key or jwt policy: who are you?
       ↓
2. Look up your booking         ← which Plan are you on?
       ↓
3. Check your luggage limits    ← rate-limit / quota policy: are you within limits?
       ↓
4. Pat-down / x-ray             ← request transforms: clean up the request
       ↓
5. Walk through the tunnel      ← CALL THE BACKEND
       ↓
6. Customs on the way back      ← response transforms: clean up the response
       ↓
7. Receipt and analytics        ← logging, metrics
       ↓
Out to the lounge (back to client)
```

The key insights:
- **Order matters.** You check the ticket before checking luggage. Same in Gravitee — auth before rate-limit before backend.
- **You can do things on both ways.** Before backend (request phase) and after backend (response phase).
- **Each step is a separate "policy."** You can add or remove them in the UI/YAML.

---

## Gravitee vs Istio — the simple distinction

A hotel analogy.

**Gravitee = the front desk.**
- Checks your reservation.
- Tells you which room (which API) you can access based on your tier.
- Tracks how many room-service orders you've made (analytics per consumer).
- If you're not paying, you don't get in.

**Istio = the staff and pipes inside the building.**
- Once you're in, how do staff (services) talk to each other?
- How does room service get from the kitchen to your room safely (mTLS)?
- If a kitchen station is broken, can we route around it (circuit breaking)?

You need **both** for a hotel to work. Gravitee handles the customer-facing business of selling rooms; Istio handles the internal operations of running the hotel.

Concrete example:
- A request from a consumer mobile app first hits Gravitee (which checks the user's plan, key, rate limit).
- Then it goes into the cluster.
- Inside the cluster, Istio handles service-to-service stuff.

---

## The three parts of Gravitee (don't get confused by the names)

Gravitee is actually **three products** under one brand. Most people don't realize this.

### 1. **APIM** (API Management)
The main one. This is what does the "API as a product" stuff above. Has:
- The **Gateway** (the actual proxy that handles traffic)
- The **Management UI** (admin web app where you publish APIs)
- The **Developer Portal** (public web app where customers sign up)

### 2. **AM** (Access Management)
Separate product. It's a **login system** — like Keycloak or Auth0. Manages users, hands out OAuth tokens, does MFA.
- Optional. Many companies use Auth0/Okta instead and just have APIM validate the tokens.

### 3. **Cockpit**
A SaaS thing for managing multiple APIM clusters from one place. Don't worry about it.

**The short version:** when someone says "Gravitee," they usually mean APIM. AM is the optional login system.

---

## What Gravitee needs to run (the stateful stuff)

You'd run it on Kubernetes. Three databases behind it — memorize these because the SRE will be asked:

| Database | What it stores | If it dies… |
|---|---|---|
| **MongoDB** | API definitions, plans, subscriptions, applications | Can't change config; existing traffic still flows |
| **Elasticsearch** | Analytics, logs, audit trail | Dashboards go blank; traffic still flows |
| **Redis** | Rate-limit counters | Rate limiting becomes inconsistent across gateway pods |

The Gateway itself (the part actually handling traffic) is **stateless**. You can scale it horizontally with HPA, no problem.

---

## The most important policies to know

Don't try to memorize all of them. Just know these:

| Policy name | What it does | Like in code |
|---|---|---|
| **`api-key`** | Check the API key header → look up the subscription | `if (req.headers['x-api-key'] !== validKey) return 401` |
| **`jwt`** | Validate JWT (signature, issuer, expiry) | Express `express-jwt` middleware |
| **`oauth2`** | Validate OAuth2 access token | Same as JWT but might do introspection |
| **`rate-limit`** | Short-term limit (per-second/minute) | `express-rate-limit` |
| **`quota`** | Long-term limit (per-day/month) | Same idea, longer window |
| **`transform-headers`** | Add, remove, or change HTTP headers | `req.headers['x-region'] = 'eu'` |
| **`transform-body`** | Modify request/response body (JSON ↔ XML, etc.) | JSON manipulation in middleware |
| **`cache`** | Cache responses for repeated identical requests | Redis caching in your service |
| **`circuit-breaker`** | Stop calling a failing backend | Like resilience4j CircuitBreaker |
| **`groovy`** / **`javascript`** | Run custom code (escape hatch) | Custom middleware function |

Senior tip: **avoid the `groovy` / `javascript` policies**. They're the "I'll just write code" escape hatch — hard to test, easy to break. Always try to do it with first-class policies first.

---

## How to talk about Gravitee when you don't have experience

This is **the** sentence you'll say in the interview. Practice it until it's natural:

> "I haven't operated Gravitee in production. What I have run is **Istio ingress gateway**, which covers the gateway primitives — routing, JWT validation, rate limiting, observability, mTLS. The gap to Gravitee is the layer **above** that: the API as a product, with Plans and Subscriptions, the Developer Portal, AM for OAuth token issuance, and the policy chain for product-level transforms. I understand the model — it's the same model as Kong and Apigee, which your job description accepts. I'd be productive on day-to-day Gravitee config within a sprint. The deeper stuff — custom policy development, AM federation, multi-environment with Cockpit — I'd learn on the job."

Why this works:
- **Honest** — no bluffing.
- Says exactly **what you do know** (Istio).
- Says exactly **what you don't** (APIM layer).
- Shows you **researched the topic** (Kong, Apigee comparison).
- Commits to a **realistic ramp-up** (sprint, not "I'll be expert tomorrow").

---

## Common interview questions with simple answers

### Q: "What's the difference between an API gateway and a service mesh?"

> "Different layers. The API gateway is at the front door — it talks to customers, knows about Plans, API keys, subscriptions. The service mesh is inside the house — it handles how services talk to each other, mTLS, retries, load balancing. Mature setups have both — a tool like Gravitee at the edge and Istio internally."

### Q: "How would you rate-limit a partner that's hitting 10,000 req/min when they're only supposed to do 1,000?"

> "First place to look: their **Subscription** in Gravitee. The Plan attached to it defines the rate limit, and Gravitee enforces it via the `rate-limit` policy. So either their rate-limit isn't configured correctly, or they're on the wrong Plan. I'd check the gateway logs to confirm Gravitee is actually returning 429s — if they're getting through, the policy isn't applying. Common cause: a misconfigured Plan, or Redis (which holds the counters) being unhealthy."

### Q: "How does JWT auth in Gravitee differ from JWT auth in Istio?"

> "In Istio, you use `RequestAuthentication` to validate the JWT and `AuthorizationPolicy` to enforce it. In Gravitee, you create a Plan with `security: JWT` and add a `jwt` policy that validates against a JWKS URL. The big conceptual difference: **Gravitee ties the JWT to a Subscription**. The token's subject identifies an Application that's subscribed to this API's Plan. So Gravitee gives you analytics per consumer, rate limits per consumer, plan-level quotas — all impossible in Istio because Istio has no concept of a consumer."

### Q: "How would you do a canary deploy of a new API version?"

> "Two layers. Inside the mesh, Istio does the actual traffic split between v1 and v2 pods with weighted routing. At the gateway, Gravitee can either: (a) just point at the same K8s Service and let Istio do the split, or (b) use the `dynamic-routing` policy with weights at the gateway. Option (a) is cleaner because Istio is built for this with Argo Rollouts/Flagger automating the progression. Option (b) is useful if you want **per-Plan** dark launches — e.g. only Gold-tier consumers see v2."

### Q: "An API consumer reports they're getting 429s. Where do you start?"

> "Step 1: confirm it's Gravitee returning the 429 (check headers like `X-Rate-Limit-Remaining: 0`). Step 2: look up their Subscription via their API key — what Plan are they on, what's the limit? Step 3: pull the gateway analytics for their consumer ID. If they're legitimately exceeding the rate, they need a bigger Plan. If 429s are spiking across many consumers at once, suspect Redis health, not consumer behavior. **Redis is the dependency that makes per-consumer rate limiting work** — when it's sick, everything looks like consumer abuse."

---

## Things that bite you in production

Even without experience, you should know these gotchas:

1. **Redis is now critical.** If Redis dies, you have to choose: fail-open (no limit, risky) or fail-closed (everything denied, also risky). Pick deliberately, document it.

2. **Custom Groovy/JS policies are scary.** They can throw, crash the gateway, or just be slow. Always reach for a first-class policy first.

3. **API key rotation is hard.** Don't replace the key in-place — that breaks the customer's app instantly. Pattern: create a **parallel subscription** with a new key, give them 24-48h to migrate, then close the old subscription.

4. **JWKS endpoint must be reachable from gateway pods.** If your IdP's JWKS lives outside the cluster, every gateway pod needs egress access. Easy NetworkPolicy mistake.

5. **Analytics in Elasticsearch is expensive at scale.** High write rate, big indices. Set ILM (Index Lifecycle Management) policies from day one — don't let it grow forever.

6. **Two layers of rate limiting can fight each other.** If you have rate-limit at Gravitee AND at Istio (local rate limit), and they don't agree, you get confusing partial denials. Pick one layer as the source of truth for each kind of limit. (Usually: Gravitee for per-consumer/per-plan, Istio for per-pod overload protection.)

---

## The quick mental model recap

If you only remember these 6 things:

1. **Gravitee turns your APIs into products** with plans, subscriptions, and a developer portal. That's the thing it does that Istio doesn't.

2. **5 core concepts:** API (the product), Plan (the tier), Application (the consumer), Subscription (the contract between them), Policy (a step in the request flow).

3. **A request flows through Gravitee like airport security:** check ID (auth) → check ticket (Plan) → check luggage (rate limit) → transform → call backend → transform response → out.

4. **Gravitee = front desk. Istio = inside the house.** You need both for a real production system.

5. **Backed by MongoDB (config), Elasticsearch (analytics), Redis (rate-limit counters).** Three stateful dependencies to watch.

6. **You haven't used it. Be honest. Say "I've run Istio ingress, the concepts transfer, I'd be productive in a sprint."** That's a winning answer.

---

## Quick self-test (5 questions, 30s each)

1. What's the difference between Gravitee and Istio in one sentence?
2. Name the 5 core Gravitee concepts.
3. What are Gravitee's three backing data stores and what's in each?
4. Where does rate limiting happen in the policy chain — before or after the backend call?
5. How would you honestly answer "have you used Gravitee in production?"

If you can answer those out loud cleanly, you're solid for the Gravitee portion of the interview.
