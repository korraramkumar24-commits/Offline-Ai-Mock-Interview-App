# TalkReady: Offline AI Mock Interview Platform
### Phase-wise Development Guide with Separate Frontend and Backend AI Prompts

Version 1.0 | Built from *TalkReady: Offline AI Mock Interview Platform, Project Documentation v1.0*

---

## Table of Contents

1. [How to Use This README](#1-how-to-use-this-readme)
2. [What "Frontend" and "Backend" Mean in an Offline PWA](#2-what-frontend-and-backend-mean-in-an-offline-pwa)
3. [Project Overview and Feature Map](#3-project-overview-and-feature-map)
4. [Architecture Summary](#4-architecture-summary)
5. [Repository Layout and Ownership](#5-repository-layout-and-ownership)
6. [The Integration Contract (Shared by Both Developers)](#6-the-integration-contract-shared-by-both-developers)
7. [Real-Time Event Catalogue](#7-real-time-event-catalogue)
8. [Optional Runner Gateway API (Online/LAN Compiler)](#8-optional-runner-gateway-api-onlinelan-compiler)
9. [Master Context Prompts (Paste First in Every AI Session)](#9-master-context-prompts-paste-first-in-every-ai-session)
10. [Phase 0: Contract, Setup and Mock Backend](#phase-0-contract-setup-and-mock-backend)
11. [Phase 1: PWA Shell, Local Auth and Database](#phase-1-pwa-shell-local-auth-and-database)
12. [Phase 2: Voice (Speech-to-Text and Text-to-Speech)](#phase-2-voice-speech-to-text-and-text-to-speech)
13. [Phase 3: Notes, Question Engine and Offline Chat](#phase-3-notes-question-engine-and-offline-chat)
14. [Phase 4: Evaluation Engine and Interview Orchestrator](#phase-4-evaluation-engine-and-interview-orchestrator)
15. [Phase 5: Resume, Code Lab, History and Progress](#phase-5-resume-code-lab-history-and-progress)
16. [Phase 6: PDF Report, Packs, Hardening and Release](#phase-6-pdf-report-packs-hardening-and-release)
17. [Integration Test Matrix](#17-integration-test-matrix)
18. [Git Workflow and Definition of Done](#18-git-workflow-and-definition-of-done)
19. [Troubleshooting Prompts](#19-troubleshooting-prompts)

---

## 1. How to Use This README

This file is written for **two developers working in parallel**, each using an AI coding tool (GitHub Copilot Chat, Cursor, Claude, ChatGPT and similar).

| Step | Frontend developer | Backend developer |
|---|---|---|
| 1 | Read sections 2 to 8 | Read sections 2 to 8 |
| 2 | Paste the **Frontend Master Context Prompt** (section 9.1) into the AI tool | Paste the **Backend Master Context Prompt** (section 9.2) |
| 3 | For each phase, paste only the **Frontend Prompt** of that phase | For each phase, paste only the **Backend Prompt** of that phase |
| 4 | Build against the **mock backend** until the real service passes its contract tests | Implement the real service and keep it passing the shared contract tests |
| 5 | Run the **Integration Checkpoint** at the end of the phase together | Same |

Rules that keep both sides compatible:

1. **The contract file is the single source of truth.** `src/contracts/` is written in Phase 0 and changes only through a pull request that both developers approve.
2. **The frontend never imports from `src/backend/`** except through `createBackend()`. The backend never imports from `src/ui/`.
3. **Every prompt ends with acceptance criteria.** Do not move to the next phase until they pass.
4. If an AI tool invents a library or API name, check it against the official docs. Pin versions as the source documentation advises.

---

## 2. What "Frontend" and "Backend" Mean in an Offline PWA

TalkReady's defining constraint is **no network at runtime**. There is no cloud server, so the two roles are defined like this:

| Role | Owns | Runs where |
|---|---|---|
| **Frontend** | React screens, routing, theming, accessibility, charts, live captions, editor UI, install/update UX, state stores, hooks that call the API, PDF/Share UI, Capacitor shell | Main thread |
| **Backend** | Auth and crypto, Dexie database and repositories, AudioWorklet and ASR/TTS/NLP/LLM workers, question engine, scoring engine, orchestrator state machine, resume parser, code runners, report data builders, pack manager, progress and spaced repetition, connectivity probe, optional Runner Gateway | Web Workers, service layer, optional LAN server |

How they connect (this replaces HTTP in the offline path):

| Concern | Mechanism |
|---|---|
| **API integration** | Typed service interfaces in `src/contracts/` (for example `AuthApi`, `InterviewApi`). The frontend calls `backend.auth.login(...)`. Every method returns a `Promise` and throws a typed `AppError`. |
| **Database connectivity** | Only the backend touches Dexie/IndexedDB and OPFS, through repository classes. The frontend reads data only via API methods or live queries exposed by the API. |
| **Real-time communication** | A typed in-memory **EventBus** (`SPEECH_START`, `PARTIAL_TRANSCRIPT`, `ANSWER_END`, `SCORED`, and so on). The frontend subscribes with a `useBackendEvent` hook. Heavy work crosses threads as typed `postMessage` with transferable buffers. |
| **Optional network** | The only real HTTP/WebSocket path is the **Runner Gateway** (Judge0/Piston proxy) for C, C++ and Java. It is off unless the user configures a URL. |

> Important: do not add a cloud database, analytics, a CDN or remote model downloads at runtime. The CSP is `connect-src 'self'` and the optional runner URL is added only when the user configures it.

---

## 3. Project Overview and Feature Map

| ID | Feature | Backend responsibility | Frontend responsibility | Phase |
|---|---|---|---|---|
| F1 | Offline login and auth | PBKDF2 and AES-GCM key wrapping, lockout, recovery key | Login, register, lock screen, guest mode | 1 |
| F2 | Offline AI chat | Intent engine, RAG, optional LLM worker | Chat UI with streaming tokens and source chips | 3 |
| F3 | Speech-to-text | AudioWorklet, VAD, streaming ASR, term correction | Mic UI, live captions, level meter | 2 |
| F4 | Text-to-speech | Piper worker, local-voice fallback, echo control | Speaking animation, captions, speed controls | 2 |
| F5 | Resume analysis | Extract, parse, ATS score, question seeds | Upload screen, score ring, issues and skill chips | 5 |
| F6 | Question generation | Template engine, retrieval, optional LLM, validation | Setup screen, question preview | 3 |
| F7 | Answer evaluation | Deterministic scoring, STAR, coaching | Result cards, coaching, retry | 4 |
| F8 | Code compiler | Runner manager, JS, Python, SQL, remote runner, harness | CodeMirror editor, test panel, timer | 5 |
| F9 | PDF report | Report data builder, jsPDF generator | Preview, export, share | 6 |
| F10 | Interview history | Queries, filters, compare, export | List, detail, compare screens | 5 |
| F11 | Progress tracking | Stats, SM-2, badges, plan | Dashboard charts, streaks, today's plan | 5 |
| F12 | PWA support | Service worker, persistence, packs | Install button, update toast, Offline Ready badge | 1 and 6 |

Scoring formulas (fixed, do not change without a team decision):

- `final = round(100 * (0.6 * content + 0.4 * delivery))`
- `content = 0.50 * keywordCoverage + 0.30 * semantic + 0.20 * completeness`
- `delivery = 0.30 * pace + 0.30 * fillers + 0.20 * pauses + 0.10 * length + 0.10 * clarity`
- `ATS = 0.20 * structure + 0.30 * skillsMatch + 0.25 * bulletQuality + 0.15 * impact + 0.10 * hygiene`
- `coding = 0.70 * passRate + 0.15 * efficiency + 0.15 * codeQuality`

---

## 4. Architecture Summary

```
+------------------------- FRONTEND (main thread) -------------------------+
| React + TS + Tailwind | Zustand stores | Router | Chart.js | CodeMirror 6 |
+---------------------------------+----------------------------------------+
                                  |  typed API calls  +  EventBus events
+---------------------------------v----------------------------------------+
| BACKEND SERVICE LAYER  createBackend(): BackendApi                        |
| Auth | Voice | Notes | Questions | Interview | Chat | Resume | Code      |
| History | Progress | Report | Packs | Settings                           |
+-------+----------+-----------+-----------+------------+------------------+
        |          |           |           |            |
   ASR worker  TTS worker  NLP worker  LLM worker  Runner workers (one/run)
        |          |           |           |            |
+-------v----------v-----------v-----------v------------v------------------+
| DATA: Dexie/IndexedDB (encrypted fields) | OPFS + Cache Storage (models)  |
| Web Crypto (PBKDF2, AES-GCM, SHA-256)                                     |
+---------------------------------------------------------------------------+
| PLATFORM: Service Worker (Workbox) | WebAssembly | WebGPU | Capacitor     |
+---------------------------------------------------------------------------+
          Optional LAN: Runner Gateway (Node) -> Judge0 / Piston
```

Per-answer real-time loop: Mic, then AudioWorklet (16 kHz PCM), then ASR worker (VAD, streaming decode), then `PARTIAL_TRANSCRIPT` events to the UI. After about 1.8 s of silence, `ANSWER_END` goes to the Orchestrator, then the NLP worker and Scoring Engine, then `SCORED`, then the reaction and next question through TTS, then an autosave to IndexedDB.

---

## 5. Repository Layout and Ownership

```
talkready/
  README.md
  docs/                      (architecture.md, scoring.md, notes-format.md)
  public/                    (manifest.webmanifest, icons/, wasm/, phrasebank/,
                              notes/, taxonomy/, problems/, worklets/)
  scripts/                   (fetch_models.*, build_pack.mjs, checksums.mjs)
  runner-gateway/            (optional Node server)            [BACKEND]
  src/
    contracts/               (types, API interfaces, events)   [SHARED, PR-gated]
    ui/                                                         [FRONTEND]
      app/ pages/ components/ hooks/ stores/ i18n/ styles/
    backend/                                                    [BACKEND]
      core/      (db.ts, crypto.ts, events.ts, connectivity.ts, packs.ts)
      auth/ voice/ nlp/ interview/ scoring/ chat/
      resume/ code/ report/ progress/
      mock/      (mockBackend.ts, fixtures/)    [BACKEND writes, FRONTEND consumes]
      index.ts   (createBackend)
  tests/
    contract/    (shared contract tests run against mock AND real backend)
    unit/ integration/ e2e/
```

| Folder | Owner | Can the other side edit it? |
|---|---|---|
| `src/ui/**`, `public/manifest*`, `public/icons` | Frontend | No (open a PR comment) |
| `src/backend/**`, `runner-gateway/**`, `scripts/**` | Backend | No |
| `src/contracts/**`, `tests/contract/**` | Shared | Only with both approvals |

---

## 6. The Integration Contract (Shared by Both Developers)

This is created in Phase 0. Both prompt sets reference it. Keep it small and strict.

```ts
// src/contracts/index.ts
export type ID = string;
export type Unsub = () => void;

export class AppError extends Error {
  constructor(public code:
    | 'AUTH_INVALID' | 'AUTH_LOCKED' | 'NOT_UNLOCKED' | 'NOT_FOUND'
    | 'VALIDATION' | 'MIC_DENIED' | 'MODEL_MISSING' | 'RUNNER_UNAVAILABLE'
    | 'TIMEOUT' | 'STORAGE_FULL' | 'CHECKSUM_MISMATCH' | 'INTERNAL',
    message: string) { super(message); }
}

/* ---------- Shared data types ---------- */
export interface UserProfile { id: ID; username: string; displayName: string; createdAt: number; isGuest: boolean; }
export interface WordTiming { text: string; start: number; end: number; conf?: number; }
export type QType = 'definition'|'explain'|'example'|'why'|'compare'|'scenario'
  |'fill_gap'|'rapid_fire'|'challenge'|'interview_style'|'followup'|'hr'|'resume'|'coding';
export interface Question { id: ID; type: QType; text: string; expectedKeywords: string[];
  difficulty: 1|2|3; source: 'template'|'llm'|'resume'|'jd'; topic: string; chunkId?: ID; }
export interface ContentSignals { keywordCoverage: number; semantic: number; completeness: number; }
export interface DeliverySignals { wpm: number; fillersPerMin: number; fillerMap: Record<string, number>;
  longPauses: number; seconds: number; avgConfidence: number; }
export interface ScoreBreakdown { content: number; delivery: number; final: number;
  signals: { c: ContentSignals; d: DeliverySignals }; }
export interface Coaching { wentWell: string[]; improve: string[]; missedKeywords: string[];
  structureTip: string; sayItBetter: string; drills: string[]; }
export interface AnswerRecord { id: ID; sessionId: ID; question: Question; transcript: string;
  audioMs: number; score: ScoreBreakdown; coaching: Coaching; followupOf?: ID; createdAt: number; }
export type InterviewMode = 'technical'|'hr'|'resume'|'coding'|'mixed'|'rapid';
export type InterviewStyle = 'friendly'|'standard'|'strict'|'rapid';
export interface SessionConfig { mode: InterviewMode; style: InterviewStyle; topics: string[];
  questionCount: 5|10|15; voice: boolean; noteSetId?: ID; resumeId?: ID; jdText?: string; }
export interface SessionSummary { id: ID; startedAt: number; endedAt?: number; mode: InterviewMode;
  totalScore: number; contentAvg: number; deliveryAvg: number; questionCount: number;
  status: 'in_progress'|'completed'|'abandoned'; }

/* ---------- API surface ---------- */
export interface AuthApi {
  register(u: string, p: string, displayName?: string): Promise<{ profile: UserProfile; recoveryKey: string }>;
  login(u: string, p: string): Promise<UserProfile>;
  loginWithRecoveryKey(u: string, key: string, newPassword: string): Promise<UserProfile>;
  guest(): Promise<UserProfile>;
  lock(): void;
  isUnlocked(): boolean;
  current(): UserProfile | null;
  listProfiles(): Promise<Pick<UserProfile, 'id'|'username'|'displayName'>[]>;
  lockoutRemainingMs(username: string): number;
  onLockChange(fn: (locked: boolean) => void): Unsub;
}
export interface VoiceApi {
  deviceCheck(): Promise<{ micOk: boolean; noiseDb: number; localVoices: string[]; webgpu: boolean; ramGb: number; tier: 'base'|'base-plus'|'full' }>;
  startListening(): Promise<void>;           // emits SPEECH_START / PARTIAL_TRANSCRIPT / ANSWER_END
  stopListening(): void;
  setSilenceMs(ms: number): void;
  speak(text: string, o?: { speed?: number }): Promise<void>; // emits TTS_START / TTS_END
  stopSpeaking(): void;
  setSpeedMultiplier(m: number): void;
  micLevel(fn: (rms: number) => void): Unsub;
}
export interface NotesApi {
  importFile(file: File): Promise<{ noteSetId: ID; topics: number; chunks: number; possibleQuestions: number }>;
  listNoteSets(): Promise<{ id: ID; title: string; source: string; topics: string[] }[]>;
  deleteNoteSet(id: ID): Promise<void>;
}
export interface ChatApi {
  newThread(): Promise<ID>;
  send(threadId: ID, text: string): Promise<{ messageId: ID }>;  // tokens arrive as CHAT_TOKEN events
  history(threadId: ID): Promise<{ id: ID; role: 'user'|'assistant'; content: string; sources?: string[]; createdAt: number }[]>;
  engine(): 'rules'|'llm';
}
export interface InterviewApi {
  plan(cfg: SessionConfig): Promise<{ sessionId: ID; questions: Question[] }>;
  start(sessionId: ID): Promise<void>;       // drives state machine, emits INTERVIEW_STATE etc.
  submitText(sessionId: ID, text: string): Promise<void>;  // text-mode answer
  command(sessionId: ID, c: 'repeat'|'hint'|'skip'|'pause'|'resume'|'stop'): void;
  retry(answerId: ID): Promise<AnswerRecord>;
  resumeUnfinished(): Promise<SessionSummary | null>;
  finish(sessionId: ID): Promise<SessionSummary>;
}
export interface ResumeApi {
  analyze(file: File, roleTarget?: string, jdText?: string): Promise<ResumeAnalysis>;
  list(): Promise<{ id: ID; fileName: string; atsScore: number; createdAt: number }[]>;
  get(id: ID): Promise<ResumeAnalysis>;
  remove(id: ID): Promise<void>;
}
export interface ResumeAnalysis { id: ID; atsScore: number;
  parts: { structure: number; skillsMatch: number; bulletQuality: number; impact: number; hygiene: number };
  sections: Record<string, string>; skills: { name: string; category: string }[];
  issues: { severity: 'high'|'medium'|'low'; rule: string; message: string; fix: string }[];
  missingKeywords: string[]; contact: { email?: string; phone?: string; links: string[] }; }
export interface CodeApi {
  languages(): Promise<{ id: string; label: string; mode: 'offline'|'remote'|'unavailable' }[]>;
  problems(filter?: { difficulty?: number; topic?: string }): Promise<ProblemMeta[]>;
  getProblem(id: string): Promise<Problem>;
  run(req: { language: string; code: string; stdin?: string; timeoutMs?: number }): Promise<RunResult>;
  submit(req: { problemId: string; language: string; code: string; sessionId?: ID }): Promise<CodeResult>;
  killAll(): void;
}
export interface RunResult { stdout: string; stderr: string; exitCode: number; timeMs: number; timedOut: boolean; }
export interface ProblemMeta { id: string; title: string; difficulty: 1|2|3; topics: string[]; }
export interface Problem extends ProblemMeta { statement: string; starter: Record<string, string>;
  visibleTests: { input: string; expected: string }[]; timeLimitMs: number; hints: string[]; }
export interface CodeResult { passed: number; total: number; score: number;
  perTest: { index: number; hidden: boolean; ok: boolean; actual?: string; timeMs: number }[]; timedOut: boolean; }
export interface HistoryApi {
  list(f?: { from?: number; to?: number; mode?: InterviewMode; topic?: string; minScore?: number; maxScore?: number; q?: string }, page?: { offset: number; limit: number }): Promise<SessionSummary[]>;
  detail(sessionId: ID): Promise<{ summary: SessionSummary; answers: AnswerRecord[] }>;
  compare(a: ID, b: ID): Promise<{ deltaTotal: number; byTopic: Record<string, number>; bySignal: Record<string, number> }>;
  exportCsv(sessionId?: ID): Promise<Blob>;
  remove(sessionId: ID): Promise<void>;
}
export interface ProgressApi {
  dashboard(): Promise<{ trend: { t: number; score: number }[]; topicMastery: Record<string, number>;
    radar: Record<'knowledge'|'clarity'|'pace'|'fillers'|'structure', number>; streakDays: number;
    weakest: string[]; todaysPlan: { dueCards: number; suggestion: string } }>;
  dueCards(limit?: number): Promise<{ id: ID; question: Question }[]>;
  reviewCard(cardId: ID, score: number): Promise<void>;
  badges(): Promise<{ key: string; label: string; awardedAt: number }[]>;
}
export interface ReportApi {
  buildSessionPdf(sessionId: ID): Promise<Blob>;
  buildProgressPdf(): Promise<Blob>;
  save(blob: Blob, filename: string): Promise<void>;   // download / share / Capacitor FS
}
export interface PackApi {
  list(): Promise<{ id: string; kind: string; version: string; size: number; installedAt: number }[]>;
  importFile(file: File): Promise<void>;     // emits PACK_PROGRESS events
  remove(id: string): Promise<void>;
  readiness(): Promise<{ offlineReady: boolean; missing: string[] }>;
  storage(): Promise<{ usage: number; quota: number; persisted: boolean }>;
  requestPersistence(): Promise<boolean>;
}
export interface SettingsApi {
  get(): Promise<Settings>; update(p: Partial<Settings>): Promise<Settings>;
  exportData(passphrase: string): Promise<Blob>; importData(file: File, passphrase: string): Promise<void>;
  eraseAll(): Promise<void>;
}
export interface Settings { silenceMs: number; ttsSpeed: number; fontScale: number; highContrast: boolean;
  recordAudio: boolean; runnerUrl?: string; language: 'en'|'hi'|'te'; autoLockMin: number; }

export interface BackendApi { auth: AuthApi; voice: VoiceApi; notes: NotesApi; chat: ChatApi;
  interview: InterviewApi; resume: ResumeApi; code: CodeApi; history: HistoryApi;
  progress: ProgressApi; report: ReportApi; packs: PackApi; settings: SettingsApi;
  events: EventBus; }
```

Factory used by the frontend: `createBackend({ useMock?: boolean }): BackendApi`.

---

## 7. Real-Time Event Catalogue

All events are typed in `src/contracts/events.ts`. The frontend subscribes; only the backend emits (except `UI_*` events).

| Event | Payload | Emitted by | UI use |
|---|---|---|---|
| `SPEECH_START` | none | ASR worker | Mic pulse animation |
| `PARTIAL_TRANSCRIPT` | `{ text }` | ASR worker | Live captions (under 1 s latency) |
| `ANSWER_END` | `{ transcript, words, audioMs }` | ASR worker | Switch to "Thinking" |
| `MIC_LEVEL` | `{ rms }` | AudioWorklet | Level meter |
| `TTS_START` / `TTS_END` | none | TTS module | Interviewer speaking animation |
| `TTS_CAPTION` | `{ sentence }` | TTS module | Caption line per sentence |
| `INTERVIEW_STATE` | `{ sessionId, state, index, total }` | Orchestrator | Progress bar, state chip |
| `QUESTION_SHOWN` | `{ question, index }` | Orchestrator | Show question card |
| `SCORED` | `{ answerId, score, coaching }` | Scorer | Result card |
| `HINT_GIVEN` | `{ level, text }` | Orchestrator | Hint bubble |
| `SESSION_DONE` | `{ summary }` | Orchestrator | Navigate to report |
| `CHAT_TOKEN` | `{ threadId, messageId, token, done }` | Chat engine | Streaming text |
| `CODE_OUTPUT` | `{ runId, stream, chunk }` | Runner | Console output |
| `PACK_PROGRESS` | `{ packId, file, pct, stage }` | Pack manager | Import progress bar |
| `MODEL_STATUS` | `{ model, status, pct? }` | Workers | "Warming up your interviewer" |
| `LOCK_CHANGED` | `{ locked }` | Auth | Redirect to lock screen |
| `CONNECTIVITY` | `{ online, runnerReachable }` | Connectivity | Code Lab language badges |
| `ERROR` | `{ code, message }` | Any | Toast |

Rules: events are fire-and-forget and never carry secrets or the DEK. Events with big payloads use transferable buffers internally and send only summaries to the UI. The UI must unsubscribe on unmount.

---

## 8. Optional Runner Gateway API (Online/LAN Compiler)

Only needed for C, C++ and Java. This is the **only** real HTTP/WebSocket service in the project. It is a small Node 20 + Express server that fronts Judge0 or Piston in Docker.

| Method and path | Request | Response |
|---|---|---|
| `GET /v1/health` | none | `200 { ok: true, backend: 'judge0'|'piston', languages: ['c','cpp','java'] }` |
| `POST /v1/run` | `{ language, code, stdin?, timeLimitMs? }` | `200 { stdout, stderr, exitCode, timeMs, timedOut }` |
| `POST /v1/run/batch` | `{ language, code, tests: [{ input }] }` | `200 { results: RunResult[] }` |
| `WS /v1/run/stream` | `{ language, code, stdin }` | Frames `{ stream: 'stdout'|'stderr', chunk }` then `{ done: true, exitCode, timeMs }` |

Constraints: CORS allow-list comes from env `ALLOWED_ORIGINS`. Limit the body to 100 KB, apply a rate limit per IP, cap output to 64 KB and enforce a server-side CPU timeout. Language ids are mapped inside the gateway. Errors use `{ error: { code, message } }`. The client calls it only after the user sets `runnerUrl` in Settings.

---

## 9. Master Context Prompts (Paste First in Every AI Session)

### 9.1 Frontend Master Context Prompt

````text
You are a senior React and TypeScript engineer helping me build the FRONTEND of "TalkReady",
an offline-first Progressive Web App for AI mock interviews. No network may be required at runtime.

STACK: React 18, Vite, TypeScript (strict), Tailwind CSS (self-hosted, no CDN), Zustand,
React Router, Chart.js (self-hosted), CodeMirror 6, react-window, vite-plugin-pwa (Workbox).

RULES:
1. I own only src/ui/** and PWA assets (manifest, icons). Never write database, crypto, ASR,
   TTS, scoring, or parsing logic. Call the backend only through the typed interfaces in
   src/contracts/index.ts via `const backend = createBackend()` exposed by a React context
   (`useBackend()`).
2. Real-time updates come ONLY from backend.events (see the event catalogue). Write a
   `useBackendEvent(type, handler)` hook that subscribes in useEffect and unsubscribes on cleanup.
3. Never use fetch/XHR/CDN links/Google Fonts/remote images. Self-host fonts and icons.
   The only allowed network call is made by the backend's Runner Gateway; the UI never calls it.
4. Never store secrets, passwords, or the encryption key in localStorage, sessionStorage or
   Zustand persist. Zustand holds UI state only.
5. Handle every state: loading, empty, error (AppError.code), offline. Show a toast on ERROR events.
6. Accessibility: keyboard operable, aria labels, captions for all spoken text, font scaling
   (settings.fontScale), high-contrast theme, prefers-reduced-motion respected.
7. Mobile-first (360 px width), then tablet and desktop. Dark and light themes.
8. Code style: functional components, small files (<200 lines), hooks for logic, no `any`,
   data fetching in hooks, components purely presentational where possible.
9. Develop against the mock backend (createBackend({ useMock: true })) until told the real one
   passes the contract tests. Switch with env VITE_USE_MOCK=false.
10. After each task, list the files you created, how I run it, and a manual test checklist.

When something in the contract is missing or unclear, STOP and tell me the exact change to
propose to the backend developer instead of inventing endpoints.
````

### 9.2 Backend Master Context Prompt

````text
You are a senior TypeScript engineer helping me build the BACKEND of "TalkReady", an offline-first
PWA for AI mock interviews. "Backend" here means the on-device service layer: Web Workers, Dexie
(IndexedDB), OPFS, Web Crypto, WebAssembly runtimes, and one OPTIONAL Node Runner Gateway.
No network may be required at runtime.

STACK: TypeScript 5 strict, Dexie.js, Web Crypto (PBKDF2-SHA-256, AES-GCM, SHA-256),
sherpa-onnx WASM (streaming ASR, Silero VAD, Piper TTS), onnxruntime-web / transformers.js
(MiniLM or bge-small embeddings, remote models DISABLED), optional WebLLM or wllama,
pdfjs-dist, mammoth, Fuse.js, Pyodide, sql.js, jsPDF + jspdf-autotable, Vitest + fake-indexeddb.

RULES:
1. I own src/backend/**, runner-gateway/**, scripts/**. I implement exactly the interfaces in
   src/contracts/index.ts and emit exactly the events in the event catalogue. Never change a
   contract signature on my own; propose the change with the reason and a migration note.
2. All database access goes through repository classes (UserRepo, SessionRepo, AnswerRepo,
   NoteRepo, ResumeRepo, CodeRepo, ProgressRepo, ChatRepo, PackRepo). Sensitive fields
   (transcript, raw_text, source_code, feedback_json, chat content) are encrypted with the
   in-memory non-extractable AES-GCM DEK. Metadata used for indexing stays plain.
3. Heavy work runs in Web Workers with typed messages and transferable buffers. Never block
   the main thread for more than about 50 ms.
4. Scoring is deterministic: the same inputs must always give the same score. An LLM may only
   reword feedback, never change a number. Use these exact formulas: [paste Section 3 formulas].
5. No runtime network: no CDN, no remote model loading (env.allowRemoteModels=false), all WASM
   and models loaded from our origin or OPFS. Only the Runner Gateway may use the network and
   only when settings.runnerUrl is set.
6. Throw AppError with the codes from the contract. Never leak raw exceptions or secrets.
7. Every public method gets a Vitest test. Provide a contract test file that runs against BOTH
   mockBackend and the real backend (tests/contract).
8. Keep mockBackend in sync with every new method so the frontend is never blocked.
9. After each task, list files created, how to run the tests, and any contract questions.

Never store the password or the DEK. Never log transcripts, resume text or keys.
````

---

# Phase 0: Contract, Setup and Mock Backend

**Goal:** both developers can work independently from day one. **Duration:** 1 to 2 days. **Owners:** Backend (lead) with Frontend review.

### 0.1 Backend Prompt (Phase 0)

````text
TASK: Bootstrap the repository and the integration contract for TalkReady.

1. Initialise a Vite + React + TypeScript (strict) project with npm. Add dependencies and pin
   exact versions: dexie, zustand, react-router-dom, vite-plugin-pwa, workbox, chart.js,
   jspdf, jspdf-autotable, vitest, fake-indexeddb, playwright, eslint, prettier.
2. Create the folder layout: src/contracts, src/ui, src/backend/{core,auth,voice,nlp,interview,
   scoring,chat,resume,code,report,progress,mock}, tests/{contract,unit,integration,e2e},
   public/{wasm,phrasebank,notes,taxonomy,problems,worklets}, scripts, runner-gateway.
3. Write src/contracts/index.ts with all types, AppError, and the API interfaces below
   [paste Section 6 of this README], and src/contracts/events.ts with the typed EventBus
   (on/emit/off, returns an unsubscribe function) and the event catalogue [paste Section 7].
4. Write src/backend/index.ts exporting createBackend({ useMock }): BackendApi.
5. Write src/backend/mock/mockBackend.ts: a complete in-memory implementation of EVERY
   interface with realistic fixture data in src/backend/mock/fixtures (3 user profiles, 12
   questions, 5 sessions with answers, a resume analysis, 6 problems, dashboard data).
   The mock must also EMIT realistic timed events: a scripted interview that emits
   SPEECH_START, PARTIAL_TRANSCRIPT every 200 ms for 3 s, ANSWER_END, SCORED, TTS_START/END,
   INTERVIEW_STATE transitions, and CHAT_TOKEN streaming at 30 ms per token.
6. Write tests/contract/*.test.ts: parameterised suites (describe.each over [mock, real]) that
   check method existence, return shapes, error codes (for example wrong password gives
   AUTH_INVALID) and event emission. The real backend column may be marked todo for now.
7. Configure ESLint rule: files in src/ui cannot import from src/backend except
   src/backend/index.ts; files in src/backend cannot import from src/ui.
8. Add scripts: dev, build, preview, test, test:contract, lint, typecheck.
9. Add a CSP meta tag template: default-src 'self'; script-src 'self' 'wasm-unsafe-eval';
   connect-src 'self'; img-src 'self' data: blob:; worker-src 'self' blob:; style-src 'self' 'unsafe-inline'.
10. Write docs/architecture.md describing contract ownership and the PR rule.

ACCEPTANCE: `npm run typecheck && npm run test:contract` passes against the mock; the mock emits
the scripted interview events; ESLint blocks cross-layer imports; the app builds.
````

### 0.2 Frontend Prompt (Phase 0)

````text
TASK: Set up the frontend shell using the contract and mock backend (created by the backend dev).

1. In src/ui create: app/App.tsx, app/routes.tsx, app/BackendProvider.tsx (React context that
   calls createBackend({ useMock: import.meta.env.VITE_USE_MOCK !== 'false' })),
   hooks/useBackend.ts, hooks/useBackendEvent.ts (typed by event name; unsubscribes on cleanup),
   stores/ui.store.ts (theme, fontScale, toasts), components/{Button,Card,Modal,Toast,
   Spinner,EmptyState,ErrorBoundary,ProgressBar,Badge}.tsx.
2. Configure Tailwind (self-hosted) with design tokens: primary #1f4e79, surface dark #0f172a,
   spacing scale, a high-contrast variant, and `prefers-reduced-motion` utilities.
3. Build the app layout: top bar with the "Offline Ready" badge (reads backend.packs.readiness()
   and subscribes to MODEL_STATUS), a bottom tab bar on mobile (Home, Interview, Chat, Code,
   Progress, Settings) and a side nav on desktop.
4. Create placeholder routes for: /login, /home, /interview, /chat, /resume, /code, /report/:id,
   /history, /progress, /settings, /packs. Lazy-load each with React.lazy.
5. Create a global ErrorBoundary and a `useErrorToasts` hook that shows a toast for every
   ERROR event with a friendly message mapped from AppError.code.
6. Add a developer-only /dev/events page that logs every backend event live (timestamp, type,
   payload preview) so integration problems are visible.
7. Add Vitest + React Testing Library tests for useBackendEvent (subscribes, unsubscribes).

ACCEPTANCE: `npm run dev` shows the shell; navigation works on 360 px and desktop; /dev/events
prints events when the mock scripted interview is triggered from a debug button; light/dark and
font scaling work; no network request except local assets.
````

**Integration Checkpoint 0:** both run `npm run dev`. The frontend triggers the mock interview and sees events in `/dev/events`. The contract PR is merged and frozen for Phase 1.

---

# Phase 1: PWA Shell, Local Auth and Database

**Features:** F1, F12 (basics). **Duration:** Week 1.

### 1.1 Backend Prompt (Phase 1)

````text
TASK: Implement the Dexie database, repositories, crypto, and AuthApi (real, not mock).

1. src/backend/core/db.ts: Dexie database "talkready" version 1 with tables users, noteSets,
   chunks, questions, sessions, answers, codeAttempts, resumes, progress ([userId+topic]),
   srsCards, badges, chatThreads, chatMessages, packs, with indexes exactly as in this schema:
   users 'id, &username'; noteSets 'id, userId, source'; chunks 'id, noteSetId, topic';
   questions 'id, chunkId, type, difficulty, lastAskedSession'; sessions 'id, userId, startedAt,
   mode, status'; answers 'id, sessionId, questionId'; codeAttempts 'id, userId, sessionId,
   language, createdAt'; resumes 'id, userId, createdAt'; progress '[userId+topic], userId, dueAt';
   srsCards 'id, userId, dueAt'; badges 'id, userId, badgeKey'; chatMessages 'id, threadId,
   createdAt'; packs 'id, kind'. Add a migration scaffold for future versions.
2. src/backend/core/crypto.ts: deriveKEK (PBKDF2-SHA-256, 600000 iterations), wrapDEK/unwrapDEK
   (AES-GCM with random IV), encryptField/decryptField returning {iv, ct} base64, sha256Hex,
   and putEncrypted/getDecrypted helpers that encrypt a list of field names.
3. src/backend/auth/AuthService.ts implementing AuthApi:
   - register: random 16-byte salt, 256-bit DEK, wrap with password KEK, ALSO wrap with a
     recovery-key KEK (24-char random code returned once, never stored). Store wrapped keys,
     IVs, salt, iterations. Reject duplicate usernames and passwords under 8 characters
     (AppError VALIDATION).
   - login: derive KEK, unwrap; on failure throw AUTH_INVALID with the SAME message for unknown
     user and wrong password. Keep the DEK as a non-extractable CryptoKey in a module-private
     variable.
   - lockout: failed-attempt counter per username in localStorage with exponential delay
     1,2,4...60 s; while locked throw AUTH_LOCKED; expose lockoutRemainingMs.
   - lock(): discard the DEK, emit LOCK_CHANGED {locked:true}. Auto-lock after settings.autoLockMin
     minutes of inactivity or when document.visibilityState is hidden longer than 2 minutes
     (listen to a `touch()` activity ping from the UI plus visibilitychange).
   - guest(): unencrypted profile flagged isGuest, data stored with userId 'guest'.
   - loginWithRecoveryKey: unwrap with the recovery KEK, then re-wrap the DEK with the new password.
4. Repositories: UserRepo, SettingsRepo (settings stored per user), PackRepo. All repo methods
   require an unlocked session and throw NOT_UNLOCKED otherwise.
5. SettingsApi.get/update, exportData/importData (AES-GCM with passphrase-derived key, one JSON
   file with all tables), eraseAll (clear all tables, OPFS, caches, localStorage keys).
6. src/backend/core/connectivity.ts: probe(url, ms) with AbortController plus a periodic
   CONNECTIVITY event (only probes the runnerUrl if configured).
7. Service worker via vite-plugin-pwa: precache shell and wasm, registerType 'prompt',
   navigateFallback, maximumFileSizeToCacheInBytes 30 MB. Expose
   `PackApi.requestPersistence/storage/readiness` (readiness checks cache + required packs).
8. Tests (Vitest + fake-indexeddb): register/login round trip; wrong password fails with the same
   error; two users cannot read each other's rows; lockout timing (fake timers); recovery key flow;
   lock clears the key and encrypted reads then throw NOT_UNLOCKED; export/import round trip.
9. Make tests/contract auth tests pass for BOTH mock and real backends.

ACCEPTANCE: all auth tests green; no plaintext of transcript-like fields in IndexedDB (inspect
manually); app works in airplane mode for login/lock; CSP shows zero blocked requests.
````

### 1.2 Frontend Prompt (Phase 1)

````text
TASK: Build login, registration, lock screen, settings and the PWA install/update UX using ONLY
the AuthApi, SettingsApi and PackApi contract.

1. /login page: list existing profiles via backend.auth.listProfiles() as cards; password field
   with show/hide; "Sign in", "Create profile", "Continue as guest", "Forgot password?".
   Show remaining lockout time from backend.auth.lockoutRemainingMs(username) as a live countdown.
   Map AppError codes to friendly messages (AUTH_INVALID: "Wrong username or password",
   AUTH_LOCKED: "Too many attempts. Try again in {s}s").
2. Register flow (3 steps): username/display name, password with strength meter and confirmation,
   then a RECOVERY KEY screen that shows the 24-char key once, with "Copy", "Download .txt", and a
   required checkbox "I saved this key" before continuing. Warn that guest data is unencrypted.
3. Lock screen: route guard `RequireUnlocked` that redirects to /login when
   backend.auth.isUnlocked() is false or a LOCK_CHANGED {locked:true} event arrives. Send an
   activity ping (throttled to once every 15 s) on pointer/key events so auto-lock works.
4. /settings page bound to SettingsApi: silence length slider (1.0 to 3.0 s), TTS speed,
   font scale, high contrast, record audio toggle (off by default, with an explanation),
   language, auto-lock minutes, optional runner URL with a "Test connection" button
   (calls backend.settings.update then waits for a CONNECTIVITY event), Export my data
   (passphrase modal), Import data, Delete all data (type-to-confirm). Autosave with a small "Saved" indicator.
5. PWA UX: capture `beforeinstallprompt` and show an "Install TalkReady" button; iOS Safari hint
   for Add to Home Screen; a toast "New version ready. Update after this interview?" driven by the
   vite-plugin-pwa `useRegisterSW` hook; never auto-reload while an interview route is active.
6. Offline Ready badge: green when backend.packs.readiness().offlineReady is true, amber with a
   tooltip listing `missing` otherwise. Storage meter in Settings (usage/quota via packs.storage())
   with a "Protect my data" button that calls requestPersistence().
7. Add Playwright tests (Chromium, context.setOffline(true)): register, copy recovery key, lock,
   wrong password message, correct login, settings persist after reload.

ACCEPTANCE: works against the mock and the real AuthService; keyboard-only login possible;
screen-reader labels present; install prompt appears on localhost; app loads in airplane mode
after first load; no password or key ever appears in localStorage/Zustand.
````

**Integration Checkpoint 1:** switch `VITE_USE_MOCK=false`. Register, copy the recovery key, lock, log in, change settings, reload, go offline and repeat. Run `npm run test:contract` with the real backend. All auth tests must be green.

---

# Phase 2: Voice (Speech-to-Text and Text-to-Speech)

**Features:** F3, F4. **Duration:** Week 2.

### 2.1 Backend Prompt (Phase 2)

````text
TASK: Implement VoiceApi: microphone capture, VAD, streaming ASR with timestamps, TTS, echo
control and device check. All offline.

1. public/worklets/capture.js: AudioWorkletProcessor that batches 128-sample blocks into 100 ms
   mono Float32 frames at 16 kHz (resample in the worklet if AudioContext ignores sampleRate) and
   posts them with transferable buffers. Also compute RMS and emit MIC_LEVEL at 10 Hz.
2. src/backend/voice/mic.ts: getUserMedia({echoCancellation:true,noiseSuppression:true,
   autoGainControl:true,channelCount:1}); map denial to AppError MIC_DENIED; startListening,
   stopListening, micLevel(fn).
3. src/backend/voice/asr.worker.ts: load the self-hosted sherpa-onnx WASM (pin the version),
   mount model files from OPFS into the virtual FS, create Silero VAD (threshold 0.5,
   minSpeech 0.25 s, minSilence = settings.silenceMs) and an online streaming recognizer.
   Post {partial}, {final with tokens + timestamps + confidences}. Throw MODEL_MISSING with a
   clear message if the Model Pack is not installed. Emit MODEL_STATUS while loading.
4. src/backend/voice/voice.ts (VoiceApi): wire worker messages to events: SPEECH_START,
   PARTIAL_TRANSCRIPT, ANSWER_END {transcript, words: WordTiming[], audioMs}. Implement idle
   nudge hook (callback after 6 s without speech) used later by the orchestrator.
5. src/nlp/corrector.ts: fuzzy term correction using normalised Levenshtein/Jaro-Winkler against
   keywords of the active note sets (try 3, 2, 1 word spans, threshold 0.85). Unit-test with
   "poly more fism" to "polymorphism".
6. src/backend/voice/commands.ts: an utterance is a command only if it has 4 words or fewer and
   closely matches repeat/hint/skip/slower/faster/pause/resume/"I'm done"/"stop interview".
   Unit-test that "I would skip the index" is NOT a command.
7. src/backend/voice/tts.worker.ts and tts.ts: Piper through sherpa-onnx WASM; speak() splits
   text into sentences, synthesises sentence by sentence (first audio under 800 ms), varies speed
   0.95 to 1.05, applies the persistent speed multiplier, inserts 120 to 300 ms pauses, plays
   via Web Audio, emits TTS_START, TTS_CAPTION per sentence, TTS_END. Mute (stop feeding) ASR
   while speaking and wait 250 ms after playback before unmuting. Cache greeting/transition audio.
   stopSpeaking() must cancel immediately.
8. Fallback: if no Piper voice, use speechSynthesis ONLY with voices where localService === true;
   if none, resolve speak() after emitting TTS_CAPTION (captions only).
9. deviceCheck(): mic permission, 2-second noise-floor estimate in dB, local voices, WebGPU
   adapter check, navigator.deviceMemory, plus a 10-second micro-benchmark (matrix multiply +
   tiny ONNX inference) to decide tier base | base-plus | full.
10. Text-mode fallback: when mic is unavailable, InterviewApi.submitText feeds the same pipeline
    with synthetic timings (document the limitation: delivery signals are neutral).
11. Tests: WER script with fixed WAV clips fed to the ASR worker (target under 15% in quiet),
    corrector tests, command filter tests, and an echo test plan (mic muted during TTS).

ACCEPTANCE: first partial caption under 1 s; ANSWER_END fires after the configured silence;
the app never transcribes its own voice; works in airplane mode; MIC_DENIED is thrown and the
UI can fall back to text.
````

### 2.2 Frontend Prompt (Phase 2)

````text
TASK: Build the voice UI components and a "Voice Lab" page using only VoiceApi and events.

1. Component <MicButton/>: states idle, listening, processing, denied; pulses on SPEECH_START;
   large touch target; keyboard shortcut Space to toggle.
2. Component <LiveCaptions/>: subscribes to PARTIAL_TRANSCRIPT and shows text with the newest words
   highlighted; clears on ANSWER_END and displays the final transcript. aria-live="polite".
3. Component <LevelMeter/> using backend.voice.micLevel (requestAnimationFrame-throttled bars).
4. Component <InterviewerAvatar/>: animated speaking state on TTS_START/TTS_END, caption bubble
   from TTS_CAPTION (so every spoken sentence is visible), and a "Replay" button.
5. Device check wizard (/device-check): calls backend.voice.deviceCheck(); shows mic permission
   prompt help, live level meter, a "Say something" noise test, speaker test button
   (backend.voice.speak("Testing one two three")), headphone recommendation banner, local-voice
   list, detected tier with a manual override select saved via SettingsApi.
6. /voice-lab page: a free-speech playground showing live captions, a word table (word, start,
   end, confidence) after ANSWER_END, WPM and filler preview computed from `words` for display
   only (the official scoring comes from the backend), speed slider calling setSpeedMultiplier,
   and "Speak this text" textarea for TTS tests.
7. Model loading UI: subscribe to MODEL_STATUS and show a friendly "Warming up your interviewer"
   progress screen; if AppError MODEL_MISSING is thrown, show a card linking to /packs.
8. Permission-denied and no-mic states offer "Continue in text mode" and persist that choice.
9. Playwright test with a fake media stream (--use-fake-device-for-media-stream) for the
   permission flow; component tests with mock events for captions and the avatar.

ACCEPTANCE: captions update live (under 1 s with real ASR, visibly streaming with the mock);
avatar and captions in sync with TTS events; text-mode fallback works; no overlapping audio
(pressing Stop calls stopSpeaking); WCAG: captions always visible.
````

**Integration Checkpoint 2:** speak into the Voice Lab. Captions must stream live and the final transcript with word timings must appear. TTS must play without the mic transcribing it. Test again in airplane mode.

---

# Phase 3: Notes, Question Engine and Offline Chat

**Features:** F2, F6. **Duration:** Week 3.

### 3.1 Backend Prompt (Phase 3)

````text
TASK: Implement NotesApi, the question engine, retrieval and ChatApi.

1. src/backend/nlp/chunker.ts: split Markdown by headings then paragraphs, merge to 80 to 200
   words, keep heading_path and topic. Parse YAML front matter (title, level, language).
2. src/backend/nlp/keywords.ts: stop-word removal, RAKE and TF-IDF across the note set, top 6 to
   10 phrases per chunk, definition detection ("X is a", "X refers to", "X means").
3. src/backend/nlp/embed.worker.ts: transformers.js / onnxruntime-web with env.allowRemoteModels
   = false and local model path; embeddings stored as Float32Array in the chunk table;
   cosine() utility; retrieval.ts returning top-k chunks. Batch embedding at import with
   progress events (MODEL_STATUS).
4. NotesApi.importFile: accept .md and .txt (and .pdf/.docx via the resume extractors in Phase 5);
   respond with { noteSetId, topics, chunks, possibleQuestions }. Handle empty, short and badly
   formatted files with VALIDATION errors. Ship prebuilt notes (java-basics, python-basics, dbms,
   os-basics, hr-common, communication-basics) preloaded as note sets on first run.
5. src/backend/interview/questionEngine.ts: implement the 11 template types (definition,
   explain-simply, example, why, compare, scenario, fill-the-gap, rapid-fire, challenge,
   interview-style, follow-up) as pure functions chunk to Question|null. Add phrasebank.ts
   loading public/phrasebank/*.json with a per-session usedVariants set (never repeat a variant).
6. Selection rules in selector.ts: weight user-selected topics, extra weight for weak topics and
   due SRS cards, at least 3 types per session, difficulty ramp (definition, example/why,
   scenario/compare), exclude questions asked in the last 3 sessions. Persist questions with
   lastAskedSession. Add validation: single sentence ending with "?" or imperative, at least 2
   expected keywords present in the chunk, embedding similarity under 0.9 versus recent questions.
7. Optional LLM generation: if the AI Pack is installed, prompt with a chunk and request JSON
   {question, expected_keywords, difficulty}; validate; retry once; otherwise fall back to templates.
8. Chat: ChatApi with two engines. RuleEngine: intent classifier (explain_concept, give_example,
   compare, hr_tip, resume_tip, start_interview, open_code_lab, small_talk) using keywords plus
   embeddings, handler per intent, answers cite chunk titles. LlmEngine: WebLLM worker with a local
   appConfig, strict NOTES-only system prompt, last 6 messages, temperature 0.4, max_tokens 220.
   Stream tokens via CHAT_TOKEN events. Guardrails: strip URLs, cap length, refuse off-topic.
   Chat history stored encrypted in chat_thread/chat_message. engine() reports 'llm' only if the
   pack is installed AND the capability check passes.
9. InterviewApi.plan(cfg) builds the question list (source: notes, resume seeds when resumeId,
   JD seeds when jdText) and creates the session row with status in_progress.
10. Tests: chunker boundaries, definition detector, each question template, no-repeat rule,
    validation rejects bad LLM output, retrieval returns the expected chunk, chat falls back to the
    rule engine when the LLM is absent.

ACCEPTANCE: importing Appendix B notes yields topics and a preview count; planning 10 questions
gives 3 or more types with a ramp and no duplicates; chat answers offline with source chips and
streams tokens without freezing the UI.
````

### 3.2 Frontend Prompt (Phase 3)

````text
TASK: Build the Notes, Interview Setup and Chat screens using NotesApi, InterviewApi.plan and ChatApi.

1. /notes page: drag-and-drop and file picker for .md/.txt; progress bar from MODEL_STATUS during
   embedding; after import show the preview card "We found {topics} topics and
   {possibleQuestions} possible questions"; list of note sets (prebuilt badge) with delete.
2. /interview/setup wizard: choose mode (HR, Technical, Resume-based, Coding, Mixed, Rapid-fire),
   topics (multi-select chips from the selected note sets), style (Friendly Coach, Standard,
   Strict, Rapid-fire), question count (5/10/15), voice or text, optional resume and optional
   job-description textarea. Disable unavailable options with an explanation (for example no
   resume yet links to /resume). "Start" calls backend.interview.plan(cfg) and shows a
   question-plan preview (types, difficulty ramp bar) before navigating to /interview/:id.
3. /chat page: thread list, message list with streaming assistant bubble (append CHAT_TOKEN
   tokens; stop spinner on done), source chips under replies, an "Offline AI" badge showing
   backend.chat.engine() ('rules' or 'llm'), a microphone button reusing <MicButton/> to dictate
   (PARTIAL_TRANSCRIPT fills the input) and a speaker toggle to read replies with voice.speak.
   Quick-prompt chips: "Explain polymorphism", "How do I answer Tell me about yourself?",
   "Review my resume bullet". Handle "start interview" intent by navigating to setup when the
   assistant returns that action (the API returns it in message metadata).
4. Auto-scroll with a "jump to latest" button; virtualise long threads; keyboard: Enter sends,
   Shift+Enter newline; disable send while streaming; cancel generation button.
5. Empty/error states: no notes yet, LLM not installed (hint card linking to /packs),
   VALIDATION error for empty files.
6. Tests: mock-driven component tests for streaming bubble and preview card; Playwright test for
   the setup wizard in offline mode.

ACCEPTANCE: setup wizard produces a valid SessionConfig; plan preview matches the backend
response; streaming chat is smooth; nothing blocks the main thread during import.
````

**Integration Checkpoint 3:** import `java-basics.md`, plan a 10-question interview, then ask the chat "Explain polymorphism". Verify source chips and the engine badge, with the network disabled.

---

# Phase 4: Evaluation Engine and Interview Orchestrator

**Features:** F7 plus the full interview flow. **Duration:** Week 4 and the first half of Week 5.

### 4.1 Backend Prompt (Phase 4)

````text
TASK: Implement the deterministic scoring engine, coaching, the orchestrator state machine,
autosave, and InterviewApi (start, submitText, command, retry, resumeUnfinished, finish).

1. src/backend/scoring/delivery.ts: compute wpm, fillersPerMin (phrase-aware filler list: um, uh,
   like, you know, basically, actually, so yeah), longPauses (gaps over 2.0 s), seconds,
   avgConfidence from WordTiming[] and audioMs. Guard against empty input and tiny durations.
2. src/backend/scoring/content.ts: Porter-stemmed keyword coverage using synonyms.json,
   semantic similarity (cosine between answer and best-chunk embeddings rescaled 0.3 to 0.8 onto
   0 to 1), completeness (has definition, example, purpose). star.ts: STAR cue-phrase +
   embedding scoring that REPLACES completeness for HR questions.
3. src/backend/scoring/final.ts: implement paceScore, fillerScore, pauseScore, lengthScore (by
   question type), clarityScore and finalScore exactly as in the formulas; return signals so the
   report can explain every number. Pure functions only, no randomness, no Date.now().
4. src/backend/scoring/coaching.ts: template feedback: wentWell, improve, missedKeywords, filler
   report, structureTip, sayItBetter (best chunk summary), drills (30-second answer, no-filler
   challenge, pace control). Optional LLM reword with a 3-second timeout; on failure use the
   template. The LLM may never alter numbers.
5. src/backend/interview/orchestrator.ts: state machine WARMUP, ASK, LISTEN, THINKING, REACT,
   FOLLOWUP, HINT, NEXT, WRAPUP, PAUSED. Emit INTERVIEW_STATE and QUESTION_SHOWN on every change.
   Behaviour: warmup greeting by name; ASK speaks the opener (mic muted); LISTEN opens ASR with
   the silence setting and a 6 s idle nudge; on ANSWER_END run command check, term correction,
   scoring, autosave the AnswerRecord, REACT phrase by quality, then follow-up decision
   (very short gives "Can you give an example?"; missed keyword gives a probe; no example gives an
   example request; max 1 follow-up, 2 in Strict), hint ladder (nudge, clue, answer) when final
   score under 40, then a varied transition. Emit SCORED with score and coaching. Ignore
   ANSWER_END unless state is LISTEN. Support pause/resume/skip/repeat/stop commands.
6. followup.ts and hints.ts implementing the above; phrase selection from the phrase bank by
   style (Friendly Coach, Standard, Strict, Rapid-fire) with the usedVariants no-repeat rule.
7. Persistence: after every answer write answer row then update the session row (running totals);
   session status in_progress until finish. resumeUnfinished() returns the latest in_progress
   session; start() on it continues from the next unanswered question.
8. retry(answerId): re-ask the same question, score the new answer, return an AnswerRecord that
   includes the previous score for comparison (store as followupOf).
9. finish(): compute totals (content avg, delivery avg, total), update progress stats (EWMA
   topic mastery new=0.7*old+0.3*latest), create srs_card rows for answers under 60, evaluate
   badges, emit SESSION_DONE.
10. submitText(): feeds the same pipeline with a synthetic timing model so text-mode works.
11. Session timer: default 15-minute soft limit emits a warning event via ERROR code INTERNAL? NO:
    add a new event SESSION_WARNING proposed to the contract owner (follow the PR rule).
12. Tests: golden scoring fixtures (model answer scores high, random answer scores low, identical
    input yields identical output), delivery edge cases, orchestrator tests with fake timers and a
    fake voice (state sequence, autosave after each answer, command vs answer, crash recovery).
13. Publish docs/scoring.md with all weights and thresholds; include a calibration script that
    compares scores with a CSV of teacher marks and reports mean absolute difference.

ACCEPTANCE: scoring is deterministic; a full 5-question text interview completes and persists;
killing the tab mid-session offers resume; every answer stores its raw signals.
````

### 4.2 Frontend Prompt (Phase 4)

````text
TASK: Build the live interview screen and the result/coaching UI driven entirely by events.

1. /interview/:sessionId page layout: top progress bar (INTERVIEW_STATE index/total), state chip
   (Asking, Listening, Thinking, Reacting, Hint), <InterviewerAvatar/>, question card
   (QUESTION_SHOWN), <LiveCaptions/>, <LevelMeter/>, and a controls bar: Repeat, Hint, Skip,
   Pause/Resume, Stop (all call backend.interview.command). In text mode show a textarea with
   "Submit answer" (backend.interview.submitText).
2. State-driven UX: Asking disables the mic and shows captions of the spoken question; Listening
   shows the pulsing mic and the idle countdown; Thinking shows a subtle "Thinking..." with
   skeleton; HINT_GIVEN shows a hint bubble with the level (nudge, clue, answer).
3. After SCORED show an animated <ScoreCard/>: final score ring, content and delivery bars, signal
   chips (WPM, fillers per minute, long pauses, keyword hits like "4 of 6"), coaching accordion
   (went well, improve, missed keywords as chips, filler report, structure tip, "Say it better"
   with a Listen button), and buttons Retry (backend.interview.retry then show a before/after
   delta with arrows) and Next. Do not block the next question; allow the card to collapse.
4. Wrap-up screen on SESSION_DONE: total score, content vs delivery, top three improvements,
   next practice goal, buttons "View report" (/report/:id), "Practice weak topics", "Home".
5. Resume banner on Home: backend.interview.resumeUnfinished() shows "Continue your interview from
   question N" and an Abandon option.
6. Handle PAUSED state (overlay), mic denied mid-session (switch to text with a toast), tab hidden
   (auto-pause suggestion), and navigation guard "Leave interview? Progress is saved".
7. Keyboard shortcuts: Space (push-to-talk toggle), H (hint), S (skip), R (repeat), Esc (pause).
8. Tests: component tests that replay a recorded event sequence (mock) and assert the UI state
   transitions; Playwright text-mode full interview with offline mode on.

ACCEPTANCE: UI never shows two states at once; every spoken sentence is captioned; results appear
within 1.5 s of ANSWER_END in the real backend; refresh mid-interview offers resume.
````

**Integration Checkpoint 4:** run a full 5-question text interview, then a voice interview. Check that scores match the formulas. Run `docs/scoring.md` example inputs and compare. Force-close the tab mid-session and verify the resume banner.

---

# Phase 5: Resume, Code Lab, History and Progress

**Features:** F5, F8, F10, F11. **Duration:** Second half of Week 5.

### 5.1 Backend Prompt (Phase 5)

````text
TASK: Implement ResumeApi, CodeApi (with runners), HistoryApi, ProgressApi and the optional
Runner Gateway.

A) RESUME (src/backend/resume)
1. extract.ts: PDF text via pdfjs-dist (self-hosted worker, rebuild lines by sorting items by y
   then x), DOCX via mammoth.extractRawText, TXT/MD native. Image-only PDFs: detect no text layer
   and return VALIDATION "OCR needed" (tesseract.js optional opt-in).
2. parse.ts: section detection (summary, education, experience, projects, skills, certifications,
   achievements), entities (email, phone, links, name, degree/year/CGPA), bullets, date ranges.
3. skills via taxonomy/skills.json (about 1000 entries with aliases) + Fuse.js fuzzy match;
   categories; evidence per skill.
4. analyze.ts: findings (length, action verbs, no metrics, weak phrases, missing sections, mixed
   date formats, missing contact) each with severity and fix; atsScore.ts with weights
   structure 20, skills 30, bullets 25, impact 15, hygiene 10. With jdText compute skill overlap
   and embedding similarity per bullet to produce missingKeywords. Store encrypted raw_text and
   parsed_json; seed questions (project role/challenge/decision/result, skill depth, JD gap
   probes) for the question engine.
5. ResumeApi.analyze/list/get/remove per the contract; optional LLM bullet rewrite (never changes
   the score).

B) CODE (src/backend/code)
1. runnerManager.ts: chooseRunner(lang) over js, py, sql, remote; languages() returns mode
   offline/remote/unavailable and reacts to CONNECTIVITY.
2. jsRunner.ts: sandboxed iframe (sandbox="allow-scripts", own CSP) hosting a fresh Worker per
   run, terminated on timeout (3 s default), output capped at 64 KB, no access to app storage.
3. pyRunner.worker.ts: Pyodide from /wasm/pyodide (self-hosted), warm worker reused, replaced
   after a timeout; capture stdout/stderr, stdin hook; first run under 2 s after warm-up.
4. sqlRunner.ts: sql.js with a fresh in-memory DB per problem from a seed script; order-insensitive
   comparison unless ORDER BY is required.
5. remoteRunner.ts: speaks to the Runner Gateway (section 8 of the README); active only if
   settings.runnerUrl is set and probe() succeeded; the CSP connect-src is extended at runtime
   only for that origin. Never send code to an unconfigured runner.
6. testHarness.ts + problems JSON in public/problems: wrap candidate function, run visible and
   hidden tests, compare output, record timings; score = 0.7*pass + 0.15*efficiency +
   0.15*quality heuristics (naming, function length, hard-coded output detection). Store
   code_attempt rows (encrypted source). Emit CODE_OUTPUT chunks for run().
7. At least 12 problems (arrays, strings, hashing, recursion, 3 SQL) with starters for JS and
   Python and hidden tests.

C) HISTORY (src/backend/history)
1. HistoryApi list with filters (date, mode, topic, score band, transcript search via an
   in-memory index built in a worker on unlock), pagination, detail, compare (delta by topic and
   signal), exportCsv, remove (cascade to answers and OPFS audio).

D) PROGRESS (src/backend/progress)
1. stats.ts: moving average over last 10 sessions, EWMA topic mastery, weekly delivery trends,
   streak calculation (local-time days), coding progress, resume score history.
2. spacedRepetition.ts: the SM-2 variant from the documentation (quality from score, ease floor
   1.3, intervals 1, 3, then interval*ease, lapses on q<3); dueCards/reviewCard.
3. badges.ts: First interview, Seven-day streak, Fillers under 3/min, Score above 80 three times,
   First coding problem solved offline. dashboard() assembles trend, topicMastery, radar
   (knowledge, clarity, pace, fillers, structure), streakDays, weakest 3, todaysPlan.

E) RUNNER GATEWAY (runner-gateway/, Node 20 + Express + ws)
1. Implement endpoints from README section 8 with zod validation, CORS allow-list, rate limit,
   body limit 100 KB, output cap 64 KB, timeouts, health endpoint, Dockerfile and
   docker-compose.yml (gateway + Judge0 or Piston). Add supertest tests and a README snippet.

Tests: parse fixtures (1-page, 2-page, multi-column), ATS golden scores, infinite loop terminates
at the limit, syntax errors reported, hidden tests honoured, SQL comparison, SM-2 table tests,
history filters, gateway contract tests. Make tests/contract pass for the real backend.

ACCEPTANCE: resume analysis under a few seconds on a mid-range device; Python/JS/SQL run in
airplane mode; C++ shows "needs a runner" when unconfigured and works through the gateway on LAN.
````

### 5.2 Frontend Prompt (Phase 5)

````text
TASK: Build the Resume, Code Lab, History and Progress screens using ResumeApi, CodeApi,
HistoryApi and ProgressApi.

A) /resume
1. Upload zone (PDF/DOCX/TXT) with role target dropdown and optional job-description textarea;
   call backend.resume.analyze; show progress and handle VALIDATION ("OCR needed") with a helpful card.
2. Result: score ring (atsScore) + five part bars (structure, skills, bullets, impact, hygiene),
   issues list grouped by severity with the suggested fix and copy button, skills chips grouped by
   category, missing keywords chips for the chosen role, contact check, previous analyses list
   with a score-history sparkline, delete button, and a primary button
   "Start resume-based interview" which pre-fills the setup wizard with resumeId.

B) /code (Code Lab)
1. Layout: problem list/filters (difficulty, topic), problem statement panel with hints (reveal
   one at a time), CodeMirror 6 editor (JS, Python, SQL, C, C++, Java modes; bundled locally; light
   and dark themes; mobile toolbar with brackets and Tab), language select showing a badge per
   language from backend.code.languages() (offline, needs-runner, unavailable) updated by
   CONNECTIVITY events with the message "C++ needs a runner. JavaScript, Python and SQL work offline."
2. Console panel streaming CODE_OUTPUT, Run button (backend.code.run with timeout), Submit button
   (backend.code.submit) showing per-test results (visible tests with actual vs expected, hidden
   tests only pass/fail), score, runtime, timed-out state, and a Stop button (killAll).
3. Countdown timer for coding-round mode; after submit when in an interview, the orchestrator will
   speak a complexity question, so the page listens to INTERVIEW_STATE and returns to
   /interview/:id.
4. Persist the draft per problem and language in component state via the API only (no
   localStorage of source code); show the Pyodide warm-up state via MODEL_STATUS.

C) /history
1. Virtualised list (react-window) with filters (date range, mode, topic, score band) and search
   box; session detail with per-question cards (transcript, scores, coaching, audio player if
   recordings exist), Retry weak questions, Export PDF, Export CSV, Delete with confirm.
2. Compare view: pick two sessions, show delta chips and a side-by-side bar chart by topic and signal.

D) /progress
1. Dashboard widgets from backend.progress.dashboard(): score line chart, topic heatmap, radar of
   five skills, streak flame, weakest three topics with "Practice now" buttons, Today's plan card
   with due flashcard count, badges grid with earned dates.
2. /flashcards: due SRS cards (dueCards), reveal answer outline, rate via reviewCard, session
   summary.
3. Charts with Chart.js (self-hosted), accessible alternatives (data table toggle), and empty
   states for new users.

Tests: component tests with mock data for each page; Playwright offline test running a Python
problem, viewing history and the dashboard.

ACCEPTANCE: all pages work against mock and real backend; code runs offline in JS/Python/SQL;
lists stay smooth with 1,000 sessions; charts match stored data.
````

**Integration Checkpoint 5:** upload a sample resume and check that the ATS score matches the golden fixture. Solve a Python problem offline. Complete two interviews, compare them, and check the dashboard and flashcards. Start the Runner Gateway on a laptop, set the URL in Settings, run C++, then turn it off and confirm the graceful message.

---

# Phase 6: PDF Report, Packs, Hardening and Release

**Features:** F9, F12 (complete), packs, APK. **Duration:** Week 6.

### 6.1 Backend Prompt (Phase 6)

````text
TASK: Implement ReportApi, the pack manager, performance guards, the Capacitor build and CI.

1. src/backend/report/pdfReport.ts (jsPDF + jspdf-autotable, lazy-loaded): A4, 40 pt margins,
   embedded Noto Sans TTF from the bundle. Sections: cover with score ring PNG, summary (final,
   content vs delivery, duration, question count, delta vs previous session), charts (bar chart
   per question and radar via Chart.js on an OffscreenCanvas or hidden canvas exported to PNG),
   question table (autoTable), delivery analysis (WPM, filler histogram, long pauses), coding
   section, resume section (ATS score and top three fixes), coaching (top three improvements,
   next goal), appendix (full transcript, formulas and weights, app version), page numbers, footer
   "Generated offline by TalkReady". Handle 5, 15 and 50 questions with correct page breaks.
   buildProgressPdf() for long-term progress. Finish a 10-question report in under 4 s.
2. ReportApi.save(blob, name): <a download>, File System Access API when available, Web Share API
   with files on mobile, and Capacitor Filesystem + Share when running inside the APK.
3. src/backend/core/packs.ts: .trpack (ZIP) import: stream entries, verify SHA-256 for each file
   against manifest.json (and an optional signature check with a built-in public key), write to
   OPFS /packs/<id>/<version>/ using chunked writes for large files, emit PACK_PROGRESS, record in
   the pack table, request persistent storage, warm the runtime, delete partial directories on
   failure (CHECKSUM_MISMATCH / STORAGE_FULL). Side-by-side version install, delete old version only
   after a successful warm-up. PackApi.readiness() returns offlineReady only when shell, runtimes
   and base models verify. Provide a loader that mounts OPFS files for sherpa-onnx (Emscripten FS),
   transformers.js / onnxruntime-web (blob URLs), WebLLM (custom appConfig), Pyodide and sql.js.
4. scripts/fetch_models.sh and .ps1, build_pack.mjs (build .trpack with manifest and checksums),
   checksums.mjs (SHA256SUMS.txt). Never commit models to Git (.gitignore).
5. Performance guards: dynamic import() for PDF lib, Pyodide, LLM; release the LLM on low-memory
   devices when leaving chat; 15-minute session soft limit; Battery Status API warning if present.
6. Security hardening: verify CSP, no analytics, npm audit, pinned versions, THIRD_PARTY.md with
   licences of every library, voice and model.
7. Capacitor: capacitor.config.ts, `npx cap add android`; offline flavour without INTERNET
   permission and a runner flavour with it; build arm64-v8a only with R8; serve COOP/COEP headers
   from Capacitor's local server if threaded WASM is used.
8. .github/workflows/build-pwa.yml and build-apk.yml (Node 20, Java 17; test, build, cap sync,
   assembleRelease, checksums, release assets); keystore from a base64 secret; never commit secrets.
9. Tests: PDF generation snapshots (page count, text presence), pack import success and tamper
   failure, quota exceeded handling, offline acceptance script in Playwright (setOffline(true))
   covering login, chat, 10-question interview in text mode, Python problem, resume analysis,
   PDF export, history, progress; assert zero network requests.

ACCEPTANCE: all contract tests pass; offline acceptance test is green; tampered pack is rejected;
APK installs and runs with the INTERNET permission absent.
````

### 6.2 Frontend Prompt (Phase 6)

````text
TASK: Build the report, pack manager and release polish screens.

1. /report/:sessionId: summary header, score ring, charts (reuse components), question table,
   delivery analysis, coding and resume sections, coaching list. Buttons: "Export PDF"
   (backend.report.buildSessionPdf, show a progress spinner, then backend.report.save with a
   sensible filename like TalkReady-2026-11-25.pdf), "Share" (Web Share when available), "Print".
   Also provide a progress PDF button on /progress.
2. /packs screen: installed packs table (kind, version, size, installed date, delete), "Import
   pack" file picker (.trpack) with a per-file progress bar from PACK_PROGRESS stages
   (reading, verifying SHA-256, writing, warming up), error states for CHECKSUM_MISMATCH and
   STORAGE_FULL with clear next steps, storage meter and "Free up space", and the persistent
   storage button. Show which features each missing pack unlocks (AI Pack: chat and richer
   follow-ups; Runtime Pack: C/C++/Java).
3. First-run onboarding: Create profile, device check, choose Base or Base + AI Pack
   (auto-suggest from tier), install Model Pack, "Warming up your interviewer" progress, then the
   green Offline Ready badge. Provide a skip and finish later option.
4. Accessibility audit: Lighthouse accessibility above 90, keyboard navigation through every
   screen, focus rings, aria-live regions for captions and toasts, large-text mode, high-contrast
   mode, reduced motion. Add i18n scaffolding (en first, hi and te JSON files) with a language
   switcher.
5. Performance: route-level code splitting, lazy chart and PDF imports, skeletons, image-free UI,
   cold start under 3 s after install; verify with Lighthouse and the performance panel.
6. PWA polish: manifest with shortcuts (Start interview, Code lab), maskable icon, theme colors,
   Lighthouse PWA all passing, update flow verified, iOS Add-to-Home-Screen hint.
7. Playwright E2E (offline): the full 9-step demo script from the documentation: airplane mode,
   login, resume upload, resume-based interview, coaching and filler report, Python problem, PDF
   export, history and progress, network panel with zero requests.

ACCEPTANCE: demo script passes end to end offline; PDF downloads and opens; pack import shows
accurate progress and rejects tampered files; Lighthouse PWA and accessibility targets met.
````

**Integration Checkpoint 6 (Release):** on a real low-end phone and a mid-range phone, run the offline acceptance test and the 5-minute demo script. Tag `v1.0.0` and verify the APK, the packs and `SHA256SUMS.txt` appear on the release.

---

## 17. Integration Test Matrix

Run this table at the end of each phase and before the release. Both developers must sign off.

| # | Scenario | Frontend verifies | Backend verifies | Pass condition |
|---|---|---|---|---|
| 1 | Register, recovery key, login | Screens, countdown | Key wrap, lockout | Same error for wrong user or password; DEK never persisted |
| 2 | Lock and auto-lock | Redirect on `LOCK_CHANGED` | DEK discarded | Encrypted reads throw `NOT_UNLOCKED` after lock |
| 3 | Live captions | Streaming text under 1 s | ASR worker partials | First partial under 1 s |
| 4 | TTS plus echo | Captions in sync | Mic muted during playback | App never transcribes itself |
| 5 | Notes import | Preview counts | Chunks, keywords, embeddings | Preview equals DB counts |
| 6 | Question planning | Plan preview | Selector rules | 3 or more types, ramp, no repeats |
| 7 | Full interview (voice) | State machine UI | Orchestrator and scoring | Same input gives same score; autosave per answer |
| 8 | Crash recovery | Resume banner | `resumeUnfinished` | Continues from the next question |
| 9 | Chat streaming | Token bubble | `CHAT_TOKEN` events | UI stays smooth; sources shown |
| 10 | Resume analysis | Score, issues, chips | Parser and ATS | Golden fixture score matches |
| 11 | Code run (Python offline) | Console and results | Pyodide worker | Infinite loop stops at the limit |
| 12 | Remote runner | Badge and message | Gateway and probe | Graceful message when unreachable |
| 13 | History and compare | Filters and charts | Queries and deltas | Chart values equal the DB |
| 14 | Progress and SRS | Dashboard and cards | SM-2 and stats | Due cards appear in the next session |
| 15 | PDF export | Progress UI, save | jsPDF builder | Opens in a common viewer; page breaks correct |
| 16 | Pack import | Progress UI | SHA-256 verify | Tampered pack rejected |
| 17 | Airplane mode (all of the above) | Full flow | Zero network requests | Playwright offline suite green |

---

## 18. Git Workflow and Definition of Done

- Branches: `main` is always buildable. Use `feature/fe-<name>` and `feature/be-<name>` branches with pull requests. Contract changes use `contract/<name>` and need both approvals.
- Commit messages: `feat(be/auth): ...`, `fix(fe/chat): ...`.
- **Definition of Done (per task):** typecheck and lint pass; unit tests added; contract tests pass for mock and real; no new network request; accessibility checked; docs updated; the mock is updated if the contract changed.
- Large models are never committed. `scripts/fetch_models.*` downloads them at build time.
- Keep `THIRD_PARTY.md` current with every library, voice and model licence.

---

## 19. Troubleshooting Prompts

Paste these into your AI tool when something breaks. Always include the exact error text, the file involved and what you expected.

**Frontend: event not reaching the UI**
````text
My React hook useBackendEvent('PARTIAL_TRANSCRIPT') never fires with the real backend but works
with the mock. Here is my hook, the provider, and the event type definition. Check for: wrong event
name, the handler captured in a stale closure, missing unsubscribe, the provider creating two
backend instances, and the event bus being emitted from a worker without being relayed to the main
thread. Propose the smallest fix and a test.
````

**Backend: worker or WASM load failure**
````text
My ASR worker fails to load sherpa-onnx WASM offline ("failed to fetch" or MIME error). Here is the
file layout, vite config and service worker config. Diagnose: wrong public path, missing .wasm in the
Workbox glob, maximumFileSizeToCacheInBytes too small, wasm MIME type, COOP/COEP headers for threaded
builds, or model files not mounted into the Emscripten FS. Give the minimal fix and a way to verify offline.
````

**Integration: contract mismatch**
````text
The frontend expects InterviewApi.plan to return { sessionId, questions } but the backend returns
{ id, items }. Compare these two files and the contract in src/contracts/index.ts. Tell me which side
deviates, produce the corrected implementation for that side only, and add a contract test that would
have caught this.
````

**Offline failure**
````text
My Playwright offline test (context.setOffline(true)) fails at step "start interview" with a blocked
request to <URL>. Find what makes the request (CDN, font, model URL, analytics), explain how to
self-host or disable it, and update the CSP and service worker precache accordingly.
````

**Performance**
````text
Scoring one answer takes 3 s on a mid-range phone (target under 1.5 s). Here is the profile. Suggest
optimisations that do not change scoring output: embedding cache, moving work to the NLP worker,
batching, avoiding repeated stemming, and lazy loading. Keep results deterministic and add a benchmark test.
````

---

**End of README.** TalkReady keeps the central promise of the source documentation: free, private and fully offline interview practice, with login, chat, resume analysis, a code lab, PDF reports and installable PWA support.
