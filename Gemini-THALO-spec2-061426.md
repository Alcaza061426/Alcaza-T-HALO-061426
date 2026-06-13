TECHNICAL ARCHITECTURE & SYSTEMS SPECIFICATION
System Identifier: Taiwan Holistic Analytics and Logical Operations (T-HALO)
Document Classification: Confidential — TFDA Internal Post-Market Digital Twin Regulatory Specification
Authority Reference: TFDA-PM-2026-T-HALO-0612
Technical Runtime: React 19 / Express 4 / Node.js ESM Architecture with Gemini GenAI Integration
1. Executive Summary & Vision
The Taiwan Holistic Analytics and Logical Operations (T-HALO) System represents a transformative paradigm in post-market medical device surveillance. Built to support the Taiwan Food and Drug Administration (TFDA, 衛生福利部食品藥物管理署), T-HALO serves as an active, full-stack, AI-orchestrated command portal. Its primary mandate is to ingest messy, non-standardized clinical incident sheets, clinician notes, and laboratory logs, restructure them using Google Gemini Large Language Models into a formal, structured regulatory hierarchy, and map those incidents against the official medical device license registry.
Historically, post-market surveillance (PMS) in medical device regulation has suffered from data fragmentation, delayed incident reports, and human bottlenecks in risk assessment. Clinician complaints are submitted hand-written or as unstructured text blocks, containing overlapping terms, missing model numbers, or non-standardized hospital names. Concurrently, matching these incidents to official import/export device licenses (such as 衛部醫器輸字 or 衛部醫器製字) requires exhaustive manual registry lookups.
T-HALO addresses these critical inefficiencies through three unified core modules:
AI Intelligence Notekeeper: A structured regulatory workspace that accepts noisy clinical inputs (via text paste or text file upload), utilizes the Gemini API (e.g., gemini-3.1-flash-lite, gemini-3.5-flash, or gemini-3.1-pro-preview) to perform an ontological data extraction, and generates detailed regulatory dossiers in Traditional Chinese (zh-TW) or English (en). It features five targeted "AI Magics" designed to execute deep causality analysis, hazard scoring, legal auditing under the Taiwan Medical Device Act, and administrative warning draft generation.
Dataset Standardizer: An intelligent ETL (Extract, Transform, Load) engine capable of translating raw, arbitrary, or semi-structured tables (CSVs, raw JSONs, XML grids, or Markdown lists) into standard JSON schemas containing device licenses and adverse event metrics, which are then synchronized with the application’s global memory store.
Recall Command Dashboard: An executive visual control center displaying key performance indicators (KPIs), calculated clinical exposure risks (such as the number of active deployed units in ICU centers throughout Taiwan), dynamic risk classification splits, symptom incidence heatmaps, and exhaustive tables tracking active device registries.
Backed by a scrolling, real-time command terminal (Syslog Monitor) and a dynamic generative system telemetry widget, T-HALO empowers TFDA officers to transition from passive, retrospective data collection to active, real-time, precision-targeted device recalls and CAPA (Corrective and Preventive Action) enforcements.
2. High-Level System Architecture
T-HALO is established as a decoupled, full-stack software container designed for stateless web-scale deployment. In development, the system runs a unified Node server file (server.ts) which boots an Express middleware listener on port 3000 and embeds a dev-configured Vite SPA instance using vite.createServer({ middlewareMode: true, appType: 'spa' }). In production, the system compiles via a dual-phase process: the Vite client assets are bundled into static CSS, JavaScript, and HTML, while server.ts is compiled via esbuild to a single standalone CommonJS bundle residing in dist/server.cjs.
The high-fidelity data ingress and outbound flow is detailed below:
code
Code
[ CLINICAL / SURVEILLANCE DATA INGRESS ]
                │
                ├──► Unstructured Adverse Incident Logs (Plaintext, TXT, MD, PDF)
                └──► Non-Standard Tabular Datasets (CSV, Semi-structured JSON, XML)
                                │
                                ▼
         ┌──────────────────────────────────────────────┐
         │             T-HALO React 19 Client           │
         │  (State Orchestrator, Sidebar Filters, UI)   │
         └──────────────┬───────────────────────────────┘
                        │
                  HTTP POST API Call
                  (JSON Payload)
                        │
                        ▼
         ┌──────────────────────────────────────────────┐
         │            T-HALO Express Server             │
         │    (Header Routing, Compression Middleware)   │
         └──────────────┬───────────────────────────────┘
                        │
             Internal Lazy SDK Instantiation
             Using process.env.GEMINI_API_KEY
                        │
                        ▼
         ┌──────────────────────────────────────────────┐
         │         Google GenAI SDK Module              │
         │     (Request Pipeline & JSON Schemas)        │
         └──────────────┬───────────────────────────────┘
                        │
                  Secure HTTPS Ingress
                  (Google API Gateway)
                        │
                        ▼
         ┌──────────────────────────────────────────────┐
         │             Gemini Large Model               │
         │ (Cognitive Reorganization & Semantic Tuning)  │
         └──────────────────────────────────────────────┘
The server architecture isolates API routes from static client-side rendering. Routing priorities on the Express backend are prioritized synchronously, placing API routing middleware before the fall-through static asset rendering or Vite development middlewares. This prevents routing collisions and ensures that network requests aimed at data processing are resolved with minimal overhead.
3. Database Domain Model & Type Specifications
To ensure strict type safety across the client-server boundary, T-HALO maintains a definitive domain model within /src/types.ts. This file governs the contract between the frontend React state, the backend JSON structures emitted by the Gemini API, and the pre-loaded dossiers.
3.1 DeviceLicense Interface
This represents the official registry record of a medical device approved for importation or domestic clinical use by the TFDA.
code
TypeScript
export interface DeviceLicense {
  licenseId: string;     // e.g., "衛部醫器輸字第012345號" or "衛部醫器製字第001294號"
  deviceName: string;    // Cleaned translation of device brand name, e.g., "愛克美" 輸液幫浦
  model: string;         // Specific hardware specification number, e.g., VG-200-PRO
  riskClass: 'Class I' | 'Class II' | 'Class III'; // Regulatory threat severity mapping
  manufacturer: string;  // Manufacturer name and country of origin, e.g., ACME Medical Systems (USA)
  components?: string[]; // Target key physical parts prone to failing, e.g., ["鋰電池", "充放電晶片"]
}
3.2 AdverseEvent Interface
The record of a single clinical incident report flagged by a Taiwan medical facility.
code
TypeScript
export interface AdverseEvent {
  eventId: string;       // Unique tracking index, e.g., AE-2026-99482
  date: string;          // ISO Date Format YYYY-MM-DD
  facility: string;      // Registry identifier of hospital department, e.g., 三軍總醫院ICU
  description: string;   // Cleaned clinician narrative detailing clinical or systemic failures
  symptoms: string[];    // Target physiological patient symptoms, e.g., ["微燒", "心跳過速"]
  severity: number;      // Calculated integer score representing severity on a 1 (low) to 10 (critical) scale
  status: 'pending' | 'reviewed' | 'resolved'; // Administrative workflow state
}
3.3 Dossier Interface
The central data capsule. It bundles a device license and its clinical adverse event reports, generating an AI-driven, structured summary.
code
TypeScript
export interface Dossier {
  id: string;                    // Primary tracking key
  title: string;                 // High-level conceptual title of safety dossier
  rawContent: string;            // The original, messy, unformatted clinical note
  organizedMarkdown: string;     // High-fidelity structured markdown parsed from Gemini
  keywords: string[];            // Scraped tags for indexing and sidebar filters
  severityScore: number;         // Calculated clinical hazard threat score (1.0 - 10.0)
  tokensPerSec?: number;         // LLM execution performance metrics
  latencySec?: number;           // LLM execution total timing
  confidencePercent?: number;    // Calculated confidence score emitted by AI output
  isArchived: boolean;           // Archive state (Active vs. Archived)
  createdAt: string;             // ISO generation timestamp
  deviceInfo?: DeviceLicense;    // Associated device metadata
  relatedEvents?: AdverseEvent[]; // Linked medical incidents
  modelUsed?: string;            // The underlying LLM engine version invoked
  aiMagicResults?: {
    [key: string]: string;       // Cached visual markdown blocks returned from AI Magics
  };
}
3.4 SysLog Interface
Maintains security-audit trace records within the interactive execution visualization dashboard.
code
TypeScript
export interface SysLog {
  id: string;
  timestamp: string;             // Client-localized time string (HH:MM:SS)
  type: 'INFO' | 'DATA' | 'AI' | 'SUCCESS' | 'WARNING'; // Semantic category styling key
  message: string;               // Text trace describing the internal execution lifecycle
}
4. Backend Server Endpoints & LLM Pipeline Specification
The backend server /server.ts is the foundational API proxy. It manages JSON validation, protects private environment credentials, parses Google Gemini SDK objects, and formats unstructured clinical text into standard schemas.
4.1 Lazy SDK Client Initialization
To protect the development container from starting crashes under missing dependencies or environment configurations, the client initialization is wrapped inside a defensive singleton instance getter:
code
TypeScript
let aiClient: GoogleGenAI | null = null;
function getGenAI(): GoogleGenAI {
  if (!aiClient) {
    const apiKey = process.env.GEMINI_API_KEY;
    aiClient = new GoogleGenAI({
      apiKey: apiKey || "MOCK_KEY_FOR_LINT",
      httpOptions: {
        headers: {
          "User-Agent": "aistudio-build",
        },
      },
    });
  }
  return aiClient;
}
4.2 Endpoint 1: System Health Monitor — GET /api/health
An endpoint utilized in automatic deployment health assessments. It determines if host configurations are active and verifies if private API keys are correctly injected within the container runtimes.
Payload Response:
code
JSON
{
  "status": "ok",
  "timestamp": "2026-06-12T18:53:59Z",
  "geminiKeyConfigured": true
}
4.3 Endpoint 2: Dossier Reorganization — POST /api/gemini/reorganize
Transforms noisy text submissions into standard clinical dossiers. It leverages structured system prompts and parses specialized highlight wrappers.
4.3.1 Architectural System Prompt
The model prompt establishes a strict persona: T-HALO, an expert medical device regulator for the TFDA Post-Market Surveillance Division. The prompt instructs the model to structure the output using five standardized markdown headings:
📋 許可證與器械對應 (License & Device Asset Mapping)
⚠️ 臨床事件敘述 (Adverse Event Narrative)
🛑 關鍵故障模式與技術診斷 (Critical Component Diagnostics & Fault Modes)
📊 TFDA 上市後風險評級 (Risk Metrics & TFDA Threat Assessment)
🛡️ 法規合規與處置指令建議 (Advisory Actions & Compliance Directive)
4.3.2 Highlight Rule Integration
To highlight critical safety indicators within the UI, the LLM is instructed to wrap key device models, warning levels, error codes, and symptoms inside standard HTML tags: <span class="coral-keyword">TARGET_KEYWORD</span>. The React frontend subsequently parses this string and dynamically applies Tailwind classes (text-[#FF6F61], bg-orange-50/100, and border-orange-100) to expose the highlighted keywords.
4.3.3 Structured Token Separation
To extract searchable keywords from the generated markdown output without triggering redundant network loops, T-HALO implements a Double-Parsing Ingress Pipeline using a custom output separator: ---KEYWORDS_SECTION---.
The model finishes generating the markdown text, appends the separator, and then appends a raw JSON array containing plain-text keyword tags (e.g., ["鋰電池", "POLY-2026", "E-401"]).
On receiving the stream, the Express server parses the text block by finding the index of the separator keyword:
code
TypeScript
const separatorIndex = fullText.indexOf("---KEYWORDS_SECTION---");
if (separatorIndex !== -1) {
  organizedMarkdown = fullText.substring(0, separatorIndex).trim();
  const keywordsPart = fullText.substring(separatorIndex + "---KEYWORDS_SECTION---".length).trim();
  // Safe JSON extraction patterns
  const arrayStart = keywordsPart.indexOf("[");
  const arrayEnd = keywordsPart.lastIndexOf("]") + 1;
  if (arrayStart !== -1 && arrayEnd !== -1) {
    keywords = JSON.parse(keywordsPart.substring(arrayStart, arrayEnd));
  }
}
If the token parser encounters irregularities, it runs a self-healing fallback parser. This parser uses a regular expression (/<span class="coral-keyword">(.*?)<\/span>/g) to extract keywords directly from the html-span tags, ensuring the application remains robust.
4.4 Endpoint 3: AI Magics Engine — POST /api/gemini/magic
Implements five clinical analysis modes on top of an existing structured markdown dossier. Based on the selected magicType, the server selects a specialized instruction set, wraps it inside an regulatory prompt template, and calls ai.models.generateContent:
SIGNAL_DETECTION (Causality Signal Diagnostic Tracker):
Identifies hidden physical component degradations and traces their impact step-by-step to explain how they cause client-facing clinical symptoms.
RECALL_RISK (Quantitative Hazard Risk Scorer):
Quantifies hazard levels on a scale of 1.0 to 10.0, evaluating demographic exposure to critical materials across hospital wards.
UDID_MAPPER (UDID Semantic Link Mapper):
Validates hardware registry conformity with standard GS1 hierarchies, mapping generic product records directly to official TFDA UDI-DI codes.
COMPLIANCE_CHECK (Regulatory Legal Compliance Audit):
Performs legal audits of manufacturer/importer actions under the Taiwan Medical Device Act, flagging report-delay infractions (such as the 7-day and 15-day reporting legal boundaries).
LETTER_DRAFT (TFDA Warning Letter & Circular Draft):
Auto-generates draft administration warnings and circular warning directives addressed to hospitals, utilizing formal legal language and detailed corrective/preventive action (CAPA) templates.
4.5 Endpoint 4: Dataset Tabular Purifier — POST /api/gemini/standardize
Resolves non-standard spreadsheet layouts without manual data cleaning. It uses the modern client configuration option responseMimeType: "application/json". The backend defines a strict JSON output schema, forcing the model to generate a clean, JSON-conforming response.
The schema requires two arrays: devices and events. This structured design allows the React client to immediately parse the response and commit the records into the active workflow memory:
code
JSON
{
  "devices": [
    {
      "licenseId": "衛部醫器輸字第012345號",
      "deviceName": "ACME Infusion Pump",
      "model": "VG-200-PRO",
      "riskClass": "Class III",
      "manufacturer": "ACME Systems Co.",
      "components": ["Lithium Cell", "Charging controller IC"]
    }
  ],
  "events": [
    {
      "eventId": "AE-2026-X884",
      "date": "2026-06-03",
      "facility": "台大醫院",
      "description": "Vessel heating thermal alarm triggered system locking",
      "symptoms": ["Fast pulse", "Dizziness"],
      "severity": 8
    }
  ]
}
5. Frontend State Architecture & Layout Components
T-HALO's client layer is written in React 19 and styled with Tailwind CSS. It uses Lucide React vector icons for its design. State is managed at the root of the application in /src/App.tsx, providing a single-source-of-truth datastore that keeps all sub-components synchronized.
code
Code
┌────────────────────────────────────────────────────────┐
       │                       src/App.tsx                      │
       │                   (Main Core State)                    │
       │                                                        │
       │       ├── dossiers: Dossier[]                          │
       │       ├── syncedDevices & syncedEvents                 │
       │       ├── language: 'zh-TW' | 'en'                     │
       │       ├── theme: 'dark' | 'light'                      │
       │       ├── activePantoneId: PantoneThemeId              │
       │       └── isProcessing: boolean                        │
       └─────┬───────────────────────────┬─────────────────────┬┘
             │                           │                     │
             ▼                           ▼                     ▼
┌─────────────────────────┐ ┌─────────────────────────┐ ┌─────────────┬───────────┐
│     src/components/     │ │     src/components/     │ │     src/components/     │
│       Sidebar.tsx       │ │     Notekeeper.tsx      │ │   DatasetStandardizer   │
└─────────────────────────┘ └─────────────────────────┘ └─────────────────────────┘
             │                           │                     │
             ▼                           ▼                     ▼
┌─────────────────────────┐ ┌─────────────────────────┐ ┌─────────────────────────┐
│     src/components/     │ │     src/components/     │ │     src/components/     │
│     RecallDashboard     │ │     ExecutionViz      │ │    /src/data/ (Mock)  │
└─────────────────────────┘ └─────────────────────────┘ └─────────────────────────┘
5.1 Real-Time Filtration Pipeline
The Sidebar and Dashboard components query the global dossier array by evaluating a combination of parameters:
Archived Status Matching: Evaluates if a dossier's isArchived property matches the state toggle showArchived.
Target Full-Text Search Queries: Checks if the search string is present in the title, rawContent, or individual keywords elements.
This localized processing ensures instantaneous grid updates without making network requests.
5.2 Responsive Layout Modules
Sidebar Layout Component (Sidebar.tsx)
A modular drawer built with vertical layout segments:
Branding Header: Highlights the T-HALO trademark and shows the active Pantone theme icon.
Core Navigation Tabs: Prominent button blocks that let users toggle the active screen state (dashboard, notekeeper, or standardizer).
Interactive Search Console: Houses the search input and the filter logic for archived cases.
Dossier Mapping List: Iterates through active items, showcasing title details, clinical danger tags, and indexing keywords.
Regulatory Preferences Panel: Includes selectors for system languages (zh-TW vs. en), theme toggles (dark vs. light), and Pantone color palette switches. It also displays a pulsing runtime indicator reflecting the current processing states (ONLINE, BUSY, or STANDBY).
AI Notekeeper Component (Notekeeper.tsx)
An interactive analysis workspace that adapts to the application state:
Messy Raw Input Mode (Active when no dossier is selected): Renders a spacious import canvas. It includes a text upload button that lets users drag and drop, read, and write plain text files (.txt, .json, .md, or .xml) using a standard FileReader interface. It also includes an action button that triggers the AI Reorganize pipeline.
Split Investigation Mode (Active when a dossier is selected): Renders a division grid with a 7:5 ratio on widescreen monitors.
Right Hand Column (5-Units span): Renders the unformatted, source clinical note inside a code-style panel (ORIGINAL_RAW_LOG), preserving the clinical context.
Left Hand Column (7-Units span): Renders the processed results across three sub-tabs:
Standard Restructured Docket Tab: Displays the generated structured markdown, using regex to replace and inject styled spans for highlighted keywords.
AI Magics Ops Tab: Contains five quick-start buttons mapped to the /api/gemini/magic pipeline. It also includes an asynchronous terminal frame that renders safety warnings and administrative letter drafts.
Conversational Grounding Chat Tab: A secure messaging interface. It enables officers to chat directly with the dossier using the active LLM engine as a reasoning partner, grounding the assistant's responses in the currently viewed safety document.
Export Engine Support: Features a print-export coordinator that dynamically opens a print frame, formats the markdown into printable HTML styles (including custom font imports for "Noto Sans TC" and "Inter"), and prompts the browser's PDF engine (window.print()).
Dataset Standardizer Component (DatasetStandardizer.tsx)
An intelligent ETL processor:
Ingress Text Area: A paste zone mapping tabular logs, CSV arrays, or irregular XML strings.
ETL Pipeline Trigger: Calls /api/gemini/standardize to process and clean the data.
Standardized Data Previewer: Displays a dual-panel spreadsheet interface:
Device License Assets Sheet: Standardizes values into formal rows tracking official license IDs, design models, risk tiers, and manufacturer origins.
Clinical Adverse Reports Sheet: Categorizes clinical values into rows tracking event IDs, dates, facility codes, descriptions, and calculated danger scores.
Global Database Synchronizer: Features a commit toggle that integrates the parsed records into the application memory. It dynamically creates corresponding dossiers for each imported event to ensure complete historical traceability.
Recall Dashboard Component (RecallDashboard.tsx)
The primary data visualization command center:
KPI Scorecard Matrix: Displays active dossier totals, compiled license assets, clinical event volumes, and calculated average risk scores, using alerting colors (green, amber, red) based on severity metrics.
Statistical Analysis Panels:
Classified Risk Metric Scale: Generates inline bar progress graphs using CSS classes, showing the ratio of high-alert Class III devices to lower-tier products.
Symptom Analysis Heatmap: Maps clinical incidence rates (e.g., Cardiac Pause, Fluid Interruption, and Thermal Overheat events) against calculated threat scores using scaled colored progress elements.
Interactive Registries: Features independent, search-filtered database grids tracking registered device models and chronological hospital safety reports.
Execution Visualization Telemetry (ExecutionViz.tsx)
Renders the underlying application performance metrics:
Pulsing Concentric Core Icon: Centered in the panel is an interactive telemetry node. It pulses and animates when asynchronous network requests are active, using the primary Pantone color style as an glowing asset overlay.
Active KPI Telemetry Grid: Renders current performance telemetry:
TOKENS/S: Average model response speed.
LATENCY: Average transaction execution speed.
CONFIDENCE: The safety alignment score.
Interactive Syslog Console: A command-style scrolling terminal. It consumes system logs and renders color-coded action types (INFO, DATA, AI, SUCCESS, or WARNING), letting users trace background executions and server pipelines.
6. Pantone Color Orchestration & Styling Engine
To convey a professional, authoritative, and distinctive visual atmosphere, T-HALO uses a customized Pantone Dynamic Color Orchestration System. This systems replaces generic design layouts with ten official Pantone Color of the Year selections, applying them via cohesive theme pairings:
Theme ID	Official Pantone Name	Hex Value	Primary Aesthetic Association
classic-blue	Classic Blue (19-4052)	#0F4C81	Standard administrative, reliable, regulatory strength
living-coral	Living Coral (16-1546)	#FF6F61	Dynamic, critical diagnostic alert focus
very-peri	Very Peri (17-3938)	#6667AB	Modern digital, futuristic analytical twin
ultra-violet	Ultra Violet (18-3838)	#5F4B8B	High-fidelity intelligence, diagnostic operations
emerald	Emerald (17-5641)	#009B77	Systematic safety, validation success
tangerine-tango	Tangerine Tango (17-1463)	#DD4124	Urgent safety intervention, Class III alert
sand-dollar	Sand Dollar (13-1106)	#B89B72	Minimalist layout, modern neutral tone
rose-quartz	Rose Quartz (13-1520)	#F7CAC9	Calm medical analysis, light ambient atmosphere
illuminating	Illuminating (13-0647)	#D3A400	Bright warning, search illumination
forest-biome	Forest Biome (19-5220)	#18453B	Deep focus, clinical surveillance research
Theme Injection Mechanics
Rather than storing style parameters in static files, T-HALO binds colors dynamically. Theme options are declared in objects (/src/data/pantoneThemes.ts) and managed via state (activePantone). The system maps the active configuration to the interface in two ways:
Tailwind Utility Intersections: Background and border highlights use responsive classes declared in the Pantone theme (e.g., bgAccent: "bg-blue-50/100 dark:bg-blue-950/20", borderAccent: "border-blue-200 dark:border-blue-800/60"). These adapt automatically to theme states.
Dynamic Style Binding: Elements that require precise matching (such as active state tags, pulsing icons, processing sliders, and call-to-action buttons) use inline style overrides to bind Hex values directly (e.g., style={{ backgroundColor: activePantone.primary }}).
System Theme Coordination
T-HALO integrates a React side-effect hook that synchronizes system theme states with document root containers:
code
TypeScript
useEffect(() => {
  const rootElement = window.document.documentElement;
  if (theme === 'dark') {
    rootElement.classList.add('dark');
    rootElement.classList.remove('light');
  } else {
    rootElement.classList.add('light');
    rootElement.classList.remove('dark');
  }
}, [theme]);
This enables seamless, real-time styling changes across the interface, using Tailwind utility states (dark:bg-slate-900, dark:text-slate-100) to maintain optimal visual contrast.
7. Operational Lifecycles & State Transitions
To understand how the application responds to user actions, we specify three core operational workflows: Note Ingestion, AI Miracle Operations, and Dataset Tabular Purge.
7.1 Note Ingestion Workflow
code
Code
[ User Inputs/Pastes Raw Clinical Data to Notekeeper ]
                       │
                       ▼
[ User clicks "AI Reorganize" with Model Select in App ]
                       │
                       ▼
[ UI sets isProcessing=True & generates a "INFO" thread to Syslog ]
                       │
                       ▼
[ Call POST /api/gemini/reorganize ] ---> [ Server parses with Gemini Model ]
                                                      │
                                                      ▼
[ Server returns Structured Markdown + Keywords array to Notekeeper ]
                                                      │
                                                      ▼
[ Client extracts Device & License information via standard Regex parsing ]
                                                      │
                                                      ▼
[ Client creates/updates Dossier entity with raw note, metadata, & metrics ]
                                                      │
                                                      ▼
[ Syslog logs "SUCCESS" event; UI sets isProcessing=False & renders structured dossier ]
7.2 AI Magic Operations Workflow
code
Code
[ User selects specific target dossier in Sidebar list ]
                       │
                       ▼
[ User switches view to "AI Magics" sub-tab in Notekeeper ]
                       │
                       ▼
[ User selects target mode (e.g., LETTER_DRAFT) & enters "Execute Special Analytics" ]
                       │
                       ▼
[ App posts organizedMarkdown context & magicType to POST /api/gemini/magic ]
                       │
                       ▼
[ Express server maps instruction schema & queries Gemini API ]
                       │
                       ▼
[ Server captures returned text blocks & evaluates execution speeds ]
                       │
                       ▼
[ Client intercepts JSON response, caches text under aiMagicResults, & maps stats on Telemetry ]
                       │
                       ▼
[ Markdown renderer converts content to UI and prints draft alert ]
7.3 Dataset Tabular Purge Workflow
code
Code
[ User imports CSV list or XML dataset file in Dataset Standardizer ]
                       │
                       ▼
[ User clicks "Purify & Clean" ] ---> [ Posts tabular data to POST /api/gemini/standardize ]
                                                      │
                                                      ▼
                                      [ Server applies JSON scheme directives ]
                                                      │
                                                      ▼
                                      [ Gemini converts data to strict JSON response ]
                                                      │
                                                      ▼
[ Client parses matching elements and renders Devices & Events preview lists ]
                       │
                       ▼
[ User selects "Commit & Inject Database" ]
                       │
                       ▼
[ Synced items are merged with global memory pools ]
                       │
                       ▼
[ App generates individual Dossiers for each new safety incident ]
8. Production Deployment, Packaging & Security Safeguards
8.1 Standalone Production Packaging
To simplify production packaging and ensure speed within serverless Cloud Run architectures, T-HALO uses an Esbuild scripting process. This compiles /server.ts into a self-contained CommonJS runtime without bundling external dependencies:
code
JSON
"build": "vite build && esbuild server.ts --bundle --platform=node --format=cjs --packages=external --sourcemap --outfile=dist/server.cjs"
During container initialization, node runs the compiled script directly (node dist/server.cjs). This bypasses ES module resolution checks, eliminates the dependency overhead of transpilation-on-the-fly utilities, and ensures sub-second startup times when scaling cold system pods.
8.2 Security & Data Privacy Safeguards
Since T-HALO processes medical device hazard logs which can contain sensitive clinician details or hospital identifiers, it implements several security measures:
Strict Server-Side Proxying: All model interactions containing Gemini API configurations run server-side. Private validation keys are never exposed to the client browser.
Defense Against Payload Vulnerabilities: Express payloads are secured with robust parser limits (limit: "50mb"). This protects endpoints against denial-of-service attempts that use massive, unstructured tabular inputs.
Audit Logs (Syslogs): Critical backend operations (such as loading raw datasets, committing data, and triggering warning letters) write structural logs to the database system. This creates a detailed audit trail of regulatory decisions.
9. Comprehensive Follow-Up Questions
To support the future physical implementation, architectural expansion, and production scaling of the T-HALO TFDA Safety Monitor, we propose twenty critical architectural, security, and algorithmic questions:
Persistent Storage Scale: Since the architecture currently relies on in-memory client state coupled with mock initialization seeds, how should we structure the transition to a durable, real-time database schema (such as Google Cloud Firestore or PostgreSQL) to sustain multi-user regulatory workflows?
Concurrency Conflict Management: In a collaborative system where multiple TFDA inspectors are evaluating the same safety dossiers, how will T-HALO prevent edit conflicts if two users run different "AI Magics" on the same record at the same time?
Strict Offline Operation Bounds: How can we adapt T-HALO's dependencies (such as fonts and Tailwind styles loaded via CDN) so that the application can run in secure, air-gapped network spaces?
Traditional Chinese Translation Quality: If clinicians mix Taiwanese local terminology (台語俗稱) with English abbreviations, how can we improve our prompt structures to make sure the model translates them accurately into formal zh-TW terms?
Structured JSON Extraction Compliance: When processing highly irregular raw input files, how can we refine the backend JSON validation rules to handle instances where the Gemini model fails to conform to standard schemas?
Extracted Keywords Indexing Capability: As our dossiers grow to contain thousands of entries, how can we implement database indexing (such as GIN Indexes in PostgreSQL) to keep keyphrase searches fast?
Data Lineage and Audit Trail Mapping: To meet legal requirements under the Taiwan Medical Device Act, how should we log and audit cases where safety values (such as Calculated Hazard Scores) are manually updated by inspectors?
Vulnerability Safeguards against AI Injection: What security measures can we implement on /api/gemini/reorganize to intercept payload injection attempts, where users might insert instructions to override regulatory threat ratings?
Integration with Hospital EMR Pipelines: What data mapping middlewares can we implement to automatically import incident reports from standard hospital formats (like HL7 or FHIR) directly into T-HALO's standardizer interfaces?
Custom Model Tuning Opportunities: To reduce operational costs, what are the technical trade-offs of using private, fine-tuned open-source models (such as LLaMA-based systems trained on TFDA historical cases) instead of external Gemini endpoints?
Comprehensive PDF Verification System: How can we verify that the generated PDFs match hospital authentication protocols, such as appending digital signatures or public cryptographic watermarks?
Interactive UI Density Optimization: To protect usability for senior TFDA officers, what layout improvements can we make to ensure high-contrast visibility when viewing dense tables on touchscreen devices?
Real-time Stream Integration support: How can we update our Express server endpoints to stream model outputs to the client in real-time using Server-Sent Events (SSE), instead of blocking UI interactions while waiting for complete responses?
Regulatory Legal Version Mapping: How can we design a database schema that matches incident dates against historical versions of the Taiwan Medical Device Act, ensuring legal citations in warning drafts remain correct based on when the event occurred?
Offline Client Cache Mechanics: How can we use Service Workers or offline storage (like IndexedDB) to save in-progress draft dossiers locally if an inspector's network connection is lost?
Performance Telemetry Calculation Accuracy: How can we refine our performance tracker to evaluate actual backend processing times, instead of using simulated confidence scores?
Dynamic Registry Synchronization Integration: How can we configure automated cron-jobs to keep T-HALO's internal license registry in sync with the official TFDA open-data registry?
Granular Access Permission Controls: How should we implement security mechanisms (such as OAuth2 and Role-Based Access Control) to restrict advanced features like drafting warning letters to authorized senior officers?
Automated Warning Letter Compliance Verification: To make sure auto-generated warning drafts are legally sound, what natural-language check systems can we implement to flag missing details like compliance timelines or legal sections?
Large Dataset Handling and Pagination: If an ingested spreadsheet contains over 100,000 rows, how can we optimize our standardizer client rendering to handle large lists without causing browser lockups or memory leakages?
