# 1) High-level product idea (single sentence)

An Expo mobile app where users pick a board/card game, open a chat with an AI that has the official rulebook loaded via a RAG pipeline, and then “chat, debate, or adjudicate” rules with the AI — with sources, rule citations, and session sharing.

---

# 2) Primary user flows

1. **Home** — grid of popular games (e.g., Chess, Catan, Uno). Each tile shows short meta: player count, average playtime, and a “Rules” badge.
2. **Game page** — short summary + buttons: “Chat”, “Upload your rulebook” (for community or custom rulebooks), “Community rulings”.
3. **Chat / Debate** — opens chatbot session. AI has pre-loaded context from the game’s rulebook. User can:

   * Ask clarifying questions (“Does this card stack?”)
   * Propose a house rule and ask for consequences/alternatives
   * Start a “formal debate” mode (AI takes a position and you take the opposing position)
4. **Session tools** — export transcript, cite sources (link to rulebook page/section), flag for moderation, save as “ruling”.
5. **Admin / Contributors** — submit new official rulebooks (PDF/URL), moderate community rulings, correct citations.

Key UX note: preload relevant rulebook chunks before showing chat UI so the bot appears responsive and starts with rule context. (Detailed RAG below.)

---

# 3) RAG / “AI has rulebook context” — recommended pipeline (technical)

Goal: make the AI reliably answer from rulebooks and cite rule sections.

1. **Ingest**

   * Accept inputs: PDFs, URLs, plain text, markdown.
   * If PDF/scan: OCR (Tesseract or commercial OCR) → clean text.
2. **Chunking**

   * Split into semantic chunks (~500–1200 tokens) with overlap (e.g., 20%). Save metadata: game, doc page, section, original text snippet, source URL/page.
3. **Embed**

   * Generate embeddings for each chunk with Gemini embedding model (Gemini embedding models are available via the Gemini API for embeddings). Use a consistent model (e.g., `gemini-embedding-001`) and store vector + metadata. ([Google AI for Developers][2])
4. **Index**

   * Upsert vectors into a vector DB (Pinecone, Qdrant, etc.) with metadata fields (game, doc_id, page, section_id).
   * (Recommendation: start with a hosted solution like Pinecone or Qdrant; both are popular choices — see comparisons). ([Medium][3])
5. **Runtime retrieval**

   * For each user message: embed the user message (same embedding model) → similarity search (top-k) → return the top N chunks (and metadata).
6. **Prompt assembly**

   * System prompt: “You are a rules adjudicator for <game>. Always prefer official rule text. When answering, cite the exact rule section and quote up to X words. If rules conflict, say so and show both passages.”
   * Build chat prompt: system + retrieved chunks (labeled with metadata) + last N turns of conversation (or a summarized version).
7. **Call LLM**

   * Send assembled prompt to Gemini chat/generation endpoint. Gemini supports chat/multi-turn; use the GenAI / Vertex/ Gemini API quickstart flow. ([Google AI for Developers][1])
8. **Return**

   * Show the answer, with inline citations (e.g., “Rulebook: p.12, Section 5.2 — ‘When a card is played…’”).
9. **Optional: grounding verification**

   * Post-check: run a short “grounding check” step that compares the LLM’s quoted text with the stored chunk to ensure the quote is verbatim (helps detect hallucinations).

---

# 4) Recommended tech stack (detailed)

* **Client:** Expo (React Native) — great for rapid mobile dev and single codebase for iOS/Android.

  * Expo auth docs and patterns to integrate OAuth/social providers are available. ([Expo Documentation][4])
* **Backend:** Node.js with Fastify or Express (Fastify recommended for speed/scalability).
* **Auth:** BetterAuth (TypeScript-native auth framework) for email/password, OAuth, 2FA, sessions. It’s a maintained project with docs and npm package. ([Better Auth][5])
* **AI / embeddings:** Google Gemini API (chat + embeddings) via Google GenAI / Vertex AI (Gemini quickstart & model reference). Gemini offers embeddings endpoints and chat generation. ([Google AI for Developers][1])
* **Vector DB (index):** Start with Pinecone or Qdrant (hosted) — both are easy to integrate and performant for RAG. You can swap later (Chroma, Supabase vector, RedisVector, Milvus, etc.). ([Medium][3])
* **Inference orchestration / LLM library:** optional — LangChain/ai-sdk integrations for easier RAG orchestration (there are Google GenAI integrations) — but you can implement a lightweight orchestration yourself. ([LangChain][6])
* **Emails:** Nodemailer (or transactional email service like SendGrid/Postmark for scale). Nodemailer is straightforward for welcome/updates. ([nodemailer.com][7])
* **Storage:** S3 (or Cloud Storage) for PDFs / originals; relational DB (Postgres) for users, games, rulings; Redis for session/cache.
* **Hosting:** Vercel / Render / Railway for backend + Cloud SQL; or Google Cloud (Vertex AI + Cloud Storage + Cloud SQL) if you want tight integration with Gemini / Vertex services. (Vertex + Gemini is a natural fit if you want managed Google infra.) ([Google Cloud][8])

---

# 5) Architecture & components (sequence)

1. **Expo app (mobile)**

   * Authentication (BetterAuth-backed endpoints)
   * Game list UI, chat UI, upload UI
2. **API Gateway / Backend (Node)**

   * Auth routes (BetterAuth)
   * Rulebook ingest endpoints (upload PDF/URL)
   * Chat/session endpoints
   * Email/webhooks (nodemailer or SendGrid)
3. **Ingestion microservice**

   * OCR, chunker, embedding job (async job queue, e.g., BullMQ)
   * Upserts into vector DB + write metadata to Postgres
4. **RAG/Chat service**

   * On chat request: embed user query → vector DB search → assemble prompt → call Gemini chat/generation endpoint → verify grounding → return response
5. **Data stores**

   * Postgres (users, games, docs, rulings)
   * Vector DB (Pinecone/Qdrant)
   * Cloud Storage (original PDFs)
   * Redis (caching, rate limiting, short-term memory)
6. **Admin UI (web)**

   * Manage games, moderate rule uploads and community rulings

Diagram (textual):
Mobile (Expo) ↔ API → (Auth + Chat)
Chat → RAG service → VectorDB + Embedding model → Gemini API → Chat response

---

# 6) Features — MVP vs v1 vs v2

MVP (2–4 weeks)

* Expo app with authentication (email + Google OAuth).
* Home with a handful of popular games.
* Backend ingestion for 3–5 rulebooks (manual upload).
* Basic RAG: chunk → embeddings → vector DB → chat with Gemini, return answers + inline citations.
* Simple transcript export and “save ruling” feature.

v1

* User uploads (PDF/URL) with OCR and chunking.
* Debate mode (AI takes a stance).
* Community rulings + voting and moderation.
* Email notifications via Nodemailer/SendGrid for rulings and updates.
* Rate limiting, usage metrics.

v2 (scale + UX)

* Support for multilingual rulebooks and multilingual embeddings (Gemini embeddings support multilingual).
* Offline on-device embeddings experiment with EmbeddingGemma or similar for privacy/low cost (Google announced on-device embedding model options — good for extreme privacy/offline). ([Google Developers Blog][9])
* Web portal for referees and tournaments.
* Pluggable policy engine for house rules and consensus documents.

---

# 7) Prompting & UX tips (to make AI behave like a referee)

* **System prompt**: be explicit — e.g., “You are an expert rules adjudicator. When citing, always attach source metadata (doc id, page, paragraph). If the rule is ambiguous, list possible interpretations and the consequences of each.”
* **Chunk presentation**: present retrieved chunks as labeled blocks `— Source: GameName / page 8 / Section 5.2 —`.
* **Ask for clarifying Qs**: if the user query is underspecified, the bot should ask 1 clarifying question before deciding.
* **Conservative fallback**: when uncertain, respond “I don’t know” or “Rules are ambiguous — here are options,” rather than fabricating exact section numbers.
* **Debate mode toggle**: let the user choose “Adjudicate” (strictly use rule text), “Explain” (friendly explanation + examples), or “Debate” (AI takes a position and argues).

---

# 8) Cost / rate-limit / privacy considerations

* **Gemini API calls & embeddings** are billed — embedding many documents can create cost; precompute embeddings during ingestion to avoid repeated cost. (Gemini docs / Vertex quickstart outline API and auth.) ([Google AI for Developers][1])
* **Vector DB**: hosted DBs have their own pricing (index, storage, QPS). Choose based on expected scale.
* **Copyright**: many rulebooks are copyrighted. Be careful about storing/redistributing proprietary rulebooks. Consider:

  * Only accept user-uploaded rulebooks (explicit consent).
  * Flag commercial/publisher rulebooks and require explicit permission before making them public.
* **User PII & conversations**: treat as sensitive; provide privacy notice and retention policies.
* **Caching & prefetch**: for popular games, pre-compute top retrieved chunks and warm caches to reduce latency/cost.

---

# 9) Security & moderation

* **Auth**: use BetterAuth with OAuth + optional 2FA.
* **Rate limiting**: protect the Gemini API usage via server-side throttling and per-user quotas.
* **Moderation**: store community flags and provide human moderation UI. Use automated filters for offensive content.
* **API keys**: never embed Gemini keys in mobile app — always route via backend.

---

# 10) Implementation roadmap (6 steps — minimal, done in this conversation)

1. **MVP scoping (1 day)** — pick 3 target games and collect official rule PDFs (or paste rule text).
2. **Backend skeleton (2–3 days)** — Node + Fastify, auth (BetterAuth quickstart), file upload, DB schema (games, docs, chunks).

   * BetterAuth docs and npm are available for quick setup. ([Better Auth][5])
3. **Ingest pipeline (3–5 days)** — OCR (if needed), chunker, embedding calls to Gemini embedding endpoint, upsert to vector DB (Pinecone / Qdrant).

   * Use Gemini embeddings for chunk vectors. ([Google AI for Developers][2])
4. **RAG chat endpoint (3 days)** — embed user queries, search top-k, assemble prompt, call Gemini chat/generate model, return response with citations. Use Gemini quickstart as reference. ([Google AI for Developers][1])
5. **Expo client (3–5 days)** — auth flows, game list, chat UI, transcript save/export, email signup with Nodemailer/SendGrid. ([nodemailer.com][7])
6. **Testing & safety (ongoing)** — hallucination tests, edge-case tests (contradictory rules), user test with small group.

---

# 11) Minimal example prompt assembly (pseudo)

System:

> You are a rules adjudicator for **Settlers of Catan**. Only use the official rulebook passages shown below to support your answers. Always include source metadata (doc id, page). If the rulebook does not cover it, say “not specified” and offer reasonable house-rule options.

Retrieved chunks:

> [1] (source: Catan_Rulebook.pdf, page 12, section 3.4) — “When the robber is moved, players with >7 cards must discard half (rounded down)…”

User: “Can I trade 2 sheep for 1 wheat with another player while the robber is on the hex?”
Assistant: [Answer citing chunk, and if necessary say “not specified” and propose house rule]

---

# 12) Risks & gotchas

* **Hallucination risk** — mitigate via strict prompting + grounding verification and showing verbatim passages.
* **Token cost** — long rulebooks + long chats can blow up token usage. Summarize older chat turns into short summaries and keep only the last N turns in context.
* **Legal** — don’t publish copyrighted rulebooks without permission.

---

# 13) Quick choices I recommend (short)

* **Vector DB**: Start with Pinecone (hosted) or Qdrant (open-source hosted). ([Medium][3])
* **Embeddings**: Gemini embedding (`gemini-embedding-001`) — precompute at ingest. ([Google AI for Developers][2])
* **Auth**: BetterAuth for quick, TypeScript-friendly auth. ([Better Auth][5])
* **Mobile**: Expo with Google/Facebook Oauth flows. ([Expo Documentation][4])
* **Emails**: Start with Nodemailer for dev; move to SendGrid/Postmark for production. ([nodemailer.com][7])

---

# 14) Next steps I can do right now (pick one)

* Draft the **MVP data model** (Postgres schema for users/games/docs/chunks/rulings).
* Draft the **ingest worker code** pseudo/real (chunking + embedding + upsert).
* Create **sample system + few-shot prompts** tuned for “adjudicator” behavior.
* Sketch the **Expo chat UI** with component list and interactions.