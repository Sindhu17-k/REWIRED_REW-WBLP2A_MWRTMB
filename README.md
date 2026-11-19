# REWIRED_REW-WBLP2A_SPARKM
OpenlyVoice-first order and udhaar (credit) management for Indian kirana stores.Why we built thisKirana shopkeepers get dozens of informal WhatsApp voice notes every day in Tamil, Hindi, Kannada, Telugu, or English—like "Anna, rendu kilo arisi, oru litre oil, moonu packet milk anuppunga."Right now, they listen to the audio, keep a mental tally, calculate the bill manually, write it in one notebook, and separately update the customer's credit ledger in another. If a customer sends a second voice note changing their order, everything gets scratched out and mistakes happen.Openly turns that into one flow:
Speak -> Understand -> Verify -> Record.Shopkeepers record or upload a customer's voice note, the app transcribes it and builds a draft order against their store catalogue, supports follow-up voice corrections (e.g., "Remove the oil and make rice five kilos"), and updates the customer's credit ledger once confirmed.Architecture PrinciplesThe main design rule in this repo: AI handles language; deterministic backend code handles money.No AI price guessing: Unit prices always come from the products table in Postgres, never from the LLM.Human in the loop: The LLM only outputs structured JSON drafts. A shopkeeper has to explicitly tap Confirm before an order hits the ledger.No silent overwrites: Voice corrections only mutate active draft orders. Confirmed orders can only be reversed via an explicit audit entry.Idempotency: Confirming an order or logging a payment requires a unique idempotency_key inside an atomic database transaction so double-taps don't duplicate balances.Derived balances: We don't store a mutable balance column on customers. Outstanding credit is always computed from ledger_entries: opening_balance + credit_sales - payments - adjustments.MVP ScopeWhat works in this build:Browser audio recording & file upload (Tamil, Hindi, Kannada, Telugu, English)Speech-to-text transcription + structured item/quantity extractionAlias-based catalogue matching (e.g., "arisi", "chawal", "rice") with disambiguation prompts if multiple products match (e.g., White Sugar vs. Brown Sugar)Context-aware voice corrections on active drafts (add, remove, or change quantities)Customer khata ledger supporting credit sales and partial/full paymentsAudit trail linking every transaction to its original voice recording, transcription, and diff historyMobile-first dashboard showing daily orders, confirmed sales, pending drafts, and total outstanding creditOut of scope (do not build during the sprint):Direct WhatsApp Cloud API integration (audio is captured/uploaded in the web app)GST invoicing, supplier management, or payrollAutomated inventory tracking or delivery routingOnline payment gatewaysStackFramework: Next.js (App Router), React, TypeScript (single repo for UI + API routes—no separate FastAPI service)UI: Tailwind CSS + shadcn/uiDatabase, Auth & Storage: Supabase (PostgreSQL, Row Level Security, private voice-notes storage bucket)AI: OpenAI API (Whisper/STT + structured JSON extraction)Hosting: VercelRepo Layoutopenly/
├── app/
│   ├── dashboard/page.tsx         # Daily metrics, recent orders, quick actions
│   ├── voice-order/page.tsx       # Audio recorder/uploader & voice corrections
│   ├── orders/
│   │   ├── page.tsx               # Order list with status filters
│   │   └── [id]/page.tsx          # Draft review, manual edits & confirmation
│   ├── customers/
│   │   ├── page.tsx               # Customer list & search
│   │   └── [id]/page.tsx          # Customer khata ledger & payment form
│   ├── api/
│   │   ├── voice/                 # POST /transcribe, POST /extract
│   │   ├── orders/                # POST /drafts, PATCH /drafts/:id, POST /drafts/:id/corrections, POST /:id/confirm
│   │   ├── customers/             # GET/POST /, GET /:id/ledger
│   │   ├── ledger/                # POST /payments
│   │   └── dashboard/             # GET /summary
│   └── layout.tsx
├── components/
│   ├── dashboard/
│   ├── voice/
│   ├── orders/
│   ├── customers/
│   └── ui/
├── lib/
│   ├── supabase/                  # Client & server-side Supabase clients
│   ├── ai/                        # System prompts & JSON schema parsers
│   ├── audio/                     # Storage upload & format validation
│   ├── validation/                # Server-side catalogue & payload guards
│   └── calculations/              # Decimal-safe totals & ledger math
├── types/
│   ├── order.ts
│   ├── customer.ts
│   ├── product.ts
│   └── ledger.ts
├── supabase/
│   ├── migrations/                # Postgres schema & RLS policies
│   └── seed.sql                   # Demo shop, catalogue & sample customer (Ravi Kumar)
├── .env.local
├── package.json
└── README.md
Local Setup1. Install dependenciesgit clone <your-repo-url> openly
cd openly
npm install
2. Environment variablesCreate .env.local in the project root:NEXT_PUBLIC_SUPABASE_URL=your_supabase_project_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key

# Server-only keys (never import in client components)
SUPABASE_SERVICE_ROLE_KEY=your_supabase_service_role_key
OPENAI_API_KEY=your_openai_api_key
3. Supabase setupCreate a new Supabase project.Run the SQL in supabase/migrations/ to set up all 8 tables and tenant RLS policies.Run supabase/seed.sql to load the demo store, catalogue items (Rice ₹60/kg, Oil ₹150/L, Milk ₹20/packet, White Sugar ₹45/kg, Brown Sugar ₹70/kg), and sample customer Ravi Kumar (₹200 opening balance).Create a private storage bucket named voice-notes.4. Run dev servernpm run dev
App runs at http://localhost:3000. Test using a mobile viewport in DevTools (390x844).Database SchemaAll tables enforce tenant isolation via shop_id.TableColumnsNotesshopsid, name, owner_id, phone, preferred_language, created_atWorkspace rootcustomersid, shop_id, name, phone, created_atBalance is derived, never stored hereproductsid, shop_id, name, aliases (TEXT[]), unit, unit_price (NUMERIC(12,2)), activeAliases hold local names (arisi, chawal)voice_sessionsid, shop_id, customer_id, audio_path, transcription, language, intent, processing_status, created_atLinks audio files to extracted intentsordersid, shop_id, customer_id, voice_session_id, status, payment_type, total_amount, created_at, confirmed_atStatus: draft, confirmed, completed, cancelledorder_itemsid, order_id, product_id, quantity (NUMERIC(12,3)), unit, unit_price, line_totalSnapshot of unit price at order timeledger_entriesid, shop_id, customer_id, order_id, type, amount, idempotency_key (UNIQUE), created_atType: credit_sale, payment, adjustment, reversalaudit_eventsid, shop_id, entity_type, entity_id, action, before_data (JSONB), after_data (JSONB), evidence_id, actor_id, created_atTracks draft diffs, approvals & reversalsAPI EndpointsMethodRouteDescriptionPOST/api/voice/transcribeUploads multipart audio to Supabase Storage and returns transcription + languagePOST/api/voice/extractParses transcription against store catalogue into structured order JSONPOST/api/orders/draftsCreates a new draft orderPATCH/api/orders/drafts/:idUpdates draft items, quantities, or selected customer manuallyPOST/api/orders/drafts/:id/correctionsSends active draft + follow-up transcription to LLM and applies validated diffPOST/api/orders/:id/confirmAtomically marks order confirmed and inserts ledger_entries row if payment_type === 'credit'GET/api/ordersLists orders (filterable by status and date)GET/api/orders/:idReturns order details, line items, and audio playback URLGET / POST/api/customersSearches customers by name/phone or creates a new customerGET/api/customers/:id/ledgerReturns chronological ledger entries and computed outstanding balancePOST/api/ledger/paymentsRecords a partial or full customer paymentGET/api/dashboard/summaryReturns today's order count, confirmed sales, pending drafts, and total credit3-Hour Build Plan (3 Developers)Lock the shared contract first (column names, API JSON payloads, status enums, .env.local keys) before splitting up.0–20 min (Setup): Init Next.js repo, run Supabase migrations + seed data, agree on TypeScript interfaces in types/.20–80 min (Parallel work):Dev 1 (Frontend): Dashboard, audio recorder/uploader UI, order review & diff view, customer khata pages.Dev 2 (Backend): Supabase queries, catalogue alias matching, decimal-safe pricing, idempotent order confirmation & ledger writes.Dev 3 (AI): /api/voice/transcribe and /api/voice/extract routes, system prompt for order extraction & contextual draft diffs.80–120 min (Integration): Wire frontend forms and recorder to backend routes and Supabase Storage.120–155 min (QA): Test edge cases—unclear speech, ambiguous items (Sugar), double-clicking Confirm, partial payments, and cross-shop RLS isolation.155–180 min (Demo prep): Re-run seed.sql for a clean state and practice the walkthrough.Demo Flow (90–120 Seconds)Uses pre-seeded customer Ravi Kumar (₹200 existing balance):Open /dashboard, show today's summary, and tap New Voice Order.Select Ravi Kumar.Record Voice Note 1: "Two kilos rice, one litre oil and three milk packets."Show the transcription and extracted draft (Rice 2 kg ₹120, Oil 1 L ₹150, Milk 3 packets ₹60 -> Total: ₹330).Record Voice Note 2 on the same draft: "Remove oil and make rice five kilos."Show the updated draft (Oil removed, Rice updated to 5 kg, new total: ₹360).Choose Credit and tap Confirm Order.Open Ravi Kumar's khata: ₹200 previous + ₹360 new credit = ₹560 outstanding.Open the order's audit/evidence view to play back both voice recordings and inspect the correction log.
