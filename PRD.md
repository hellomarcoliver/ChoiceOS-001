Build a production-ready web app called “Strategic Choice Catalog Factory” that automatically generates (via LLM “agents”) hundreds of strategic decisions (forced choices) across decision domains, subcategories, and 55 Business Model Navigator patterns. The app must run end-to-end with minimal user input and export a complete catalog as CSV/XLSX/JSON.

NON-NEGOTIABLE PRODUCT BEHAVIOR
- The app does NOT recommend choices. It enumerates decisions.
- Every decision is a forced binary / mutually exclusive choice with real sacrifices on both sides.
- If both options can coexist, discard the decision.
- If a decision is reversible within 6–12 months (for tech/product subcategories), discard it.
- The system must produce hundreds of decisions (target 300–1000). If it produces fewer than 200 valid decisions, it must automatically run additional generation passes until thresholds are met (or explicitly report which domains/subcategories could not be exhausted and why).
- The app must support running “full mode” (all 55 business model patterns) and “inferred mode” (only patterns inferred from website + inputs).
- The app must deduplicate, normalize, score completeness, and identify gaps.

USER INPUTS (Onboarding)
The user provides:
1) Company website URL (optional: paste text instead)
2) Primary customer (free text)
3) Core problem solved (free text)
Optional:
- Industry pack selector (default: High-Tech / SaaS)
- Toggle: Full 55 BMI patterns vs Inferred patterns
- Target depth: Standard (300) vs Deep (800)

OUTPUTS
- A Strategic Choice Catalog grouped by Domain → Subcategory
- Each decision includes:
  id, domain, subcategory, decisionQuestion, optionA, optionB,
  sacrificeA, sacrificeB, whyUnavoidable, irreversibility (low/med/high),
  stage (early/scaling/enterprise), generatorSource (theory/bmi/tech/competitive),
  bmiPatterns[], confidence (0–1), evidence[] (short bullets)
- Export buttons: CSV, XLSX, JSON
- UI explorer: filter by domain, subcategory, stage, irreversibility, confidence, generatorSource, status (Mandatory/Relevant/Latent/OutOfScope; Explicit/Implicit/Avoided)
- Coverage dashboard: counts per domain/subcategory, “missing areas,” “generate missing” button, and a coverage score.

ARCHITECTURE REQUIREMENTS
- Build as a Next.js (App Router) TypeScript app with a minimal clean UI.
- Use a Node/TS backend (Next API routes) with a job queue (BullMQ + Redis, or a simple in-memory queue for dev with clear interface to swap to Redis).
- Use a database (SQLite via Prisma for dev; easily switchable to Postgres) to store runs, jobs, decisions, and audit logs.
- Add authentication only if needed; otherwise single-user local is fine for MVP.
- Must be runnable locally with `npm install && npm run dev`.
- Include `.env.example` and clear README.
- Use OpenAI API for LLM calls (or a provider abstraction) with a single `LLMClient` interface so it’s easy to swap providers.
- Implement robust rate limiting, retries with exponential backoff, and job-level timeouts.
- Every generator job must validate output with a strict JSON schema; invalid outputs are rejected and auto-regenerated with corrective prompts.

AGENT PIPELINE (THE CORE)
Implement an orchestrated pipeline with four generator types + two post-processors:

1) Orchestrator Agent
- Creates a “work plan” of micro-jobs based on selected industry pack:
  a) Generator 1: Strategy Theory decomposition per Domain
  b) Generator 3: Product/Tech Reality per Subcategory
  c) Generator 2: Business Model Pattern explosion per BMI pattern
  d) Generator 4: Competitive/Empirical contrasts (predefined scenarios per industry pack)
- Determines job counts and targets:
  - Each domain: >= 30 decisions (theory)
  - Each subcategory: >= 10 decisions (tech reality)
  - Each BMI pattern: >= 15 decisions (full mode), or top inferred patterns in inferred mode
- If inferred mode: infer relevant BMI patterns from signals (pricing model, packaging, revenue logic) and run at least 8 patterns; user can expand to full 55.

2) Generator 1 — Strategy Theory Agent (per Domain)
3) Generator 2 — BMI Pattern Agent (per Pattern)
4) Generator 3 — Product & Technology Reality Agent (per Subcategory)
5) Generator 4 — Competitive / Empirical Agent (per Contrast Scenario)

6) Deduplicator + Normalizer Agent
- Canonicalize wording, normalize domain/subcategory names, merge only if decisionQuestion + sacrifices match.
- Create stable IDs.

7) Completeness Auditor Agent
- Checks domain/subcategory saturation and identifies gaps.
- Triggers targeted “top-up” jobs automatically until coverage thresholds met or max iterations reached (e.g., 3 rounds).
- Produces “Coverage Report” with:
  counts, missing areas, low-confidence clusters, and why gaps remain.

STRICT OUTPUT FORMAT FOR ALL GENERATOR AGENTS
All generator agents MUST output ONLY valid JSON (no markdown) in this schema:

{
  "decisions": [
    {
      "domain": "string",
      "subcategory": "string",
      "decisionQuestion": "string",
      "optionA": "string",
      "optionB": "string",
      "whyUnavoidable": "string",
      "sacrificeA": "string",
      "sacrificeB": "string",
      "irreversibility": "low"|"medium"|"high",
      "stage": "early"|"scaling"|"enterprise",
      "generatorSource": "StrategyTheory"|"BusinessModelPattern"|"ProductTechReality"|"CompetitiveEmpirical",
      "bmiPatterns": ["string"],
      "confidence": number,
      "evidence": ["string"]
    }
  ]
}

Any decision missing sacrifices or having non-exclusive options must be removed by validation.

INDUSTRY PACK: HIGH-TECH / SAAS (DEFAULT)
Hardcode this pack in a config file `/packs/hightech_saas.ts`:

Decision Domains:
- WINNING ASPIRATION
- WHERE TO PLAY
- HOW TO WIN
- PRODUCT SCOPE & EVOLUTION
- GROWTH MODEL & GO-TO-MARKET
- TECHNOLOGY & PLATFORM CAPABILITIES
- CUSTOMER EXPERIENCE & ADOPTION
- ORGANIZATION & TALENT
- CAPITAL, FUNDING & RESOURCE ALLOCATION
- ECOSYSTEM, PARTNERSHIPS & COMPETITION

Subcategories (Product/Tech Reality list):
- Core Architecture & System Design
- Data Architecture
- Event Model
- Multi-Tenancy Model
- Deployment Topology
- Platform & Extensibility
- API Strategy
- SDK Strategy
- Plugin / Extension Model
- Backward Compatibility
- Versioning & Lifecycle
- Feature Scope Boundaries
- Customization & Configuration
- Integration Strategy
- UX Focus
- Workflow Model
- Onboarding & Adoption
- Analytics & Intelligence
- AI & Automation
- Security, Privacy & Compliance
- Performance, Scale & Reliability
- DevOps & Delivery Model
- Observability & Incident Response
- Collaboration & Multi-User Dynamics
- Admin, Control & Governance
- Localization & Globalization
- Search, Discovery & Navigation
- Marketplace & Ecosystem Surface
- Documentation & Knowledge
- Testing, Quality & Risk

Competitive contrast scenarios:
- Product-led vs Sales-led growth
- Enterprise vs SMB focus
- Open platform vs closed ecosystem
- Opinionated workflow vs configurable platform
- Services-led delivery vs pure SaaS
- Best-of-breed vs suite strategy

BMI PATTERNS (ALL 55)
Create `/knowledge/businessModelPatterns.json` with EXACTLY these 55 pattern names:
AddOn, Affiliation, Aikido, Auction, Barter, Bundling, CashMachine, CrossSelling, Crowdfunding, Crowdsourcing,
CustomerLoyalty, Digitization, DirectSelling, ECommerce, ExperienceSelling, FlatRate, Freemium, Franchising,
HiddenRevenue, IngredientBranding, Integrator, LayerPlayer, Leasing, Licensing, LockIn, LongTail, MassCustomization,
NoFrills, OpenBusinessModel, Orchestrator, PayPerUse, PeerToPeer, PerformanceBasedContracting, Platform, RazorAndBlade,
RentInsteadOfBuy, RevenueSharing, ReverseEngineering, ReverseInnovation, RobinHood, SelfService, ShopInShop,
SolutionProvider, Subscription, Supermarket, TargetThePoor, TrashToCash, TwoSidedMarket, UltimateLuxury,
UserDesigned, WhiteLabel, Wholesaler

WEBSITE INGESTION
- Implement a basic website scraper that fetches and extracts readable text from:
  homepage, product pages, pricing pages, careers page (if present).
- If scraping fails (CORS/blocked), allow user to paste text manually.
- Store extracted text and derived signals for transparency.

DERIVED SIGNALS (for inferred mode)
Implement a lightweight signal extractor that outputs:
- businessModelGuess (services/saas/hybrid)
- likelyBMIpatterns (top 8 with confidence)
- impliedICP
- pricingSignals (flat/subscription/usage/enterprise quote)
- distributionSignals (sales-led/product-led/partner-led)
These are used only to prioritize jobs, not to recommend decisions.

UI PAGES
1) Onboarding: inputs + toggles + start button
2) Run Progress: shows queued/running/completed jobs, counts, and live decision total
3) Catalog Explorer: table + filters + grouping by domain/subcategory
4) Coverage Dashboard: charts (simple counts) + “Generate Missing Areas” button
5) Export: CSV/XLSX/JSON download
6) Run History: list previous runs, open and export

QUALITY GATES
- Validate JSON schema
- Validate decision exclusivity (heuristic: options contain opposing framing; plus a LLM “validator” pass for borderline cases)
- Validate sacrifices non-empty and non-trivial
- Enforce minimum decision counts per job; auto-rerun with corrective prompt if insufficient
- Enforce global minimum: 300 decisions in Standard mode; 800 in Deep mode (unless user stops early)

DELIVERABLES
- Full source code
- README with setup instructions
- `.env.example`
- Prisma schema + migrations
- Sample run seed data (optional)
- Clear separation of concerns: packs, knowledge, pipeline, validators, UI

IMPLEMENTATION DETAILS
- Provide a `prompts/` directory with prompt builders for each agent type.
- Provide a `validators/` directory with: schema validator, exclusivity validator, sacrifice validator, dedupe normalizer.
- Provide `pipeline/` directory: orchestrator, job runner, retry logic, completeness loop.
- Include unit tests for validators and prompt builders.

Build it now. Do not leave TODOs. If something is ambiguous, make a reasonable assumption and document it in README.
