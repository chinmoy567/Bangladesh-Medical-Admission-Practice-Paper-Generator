# PRD — Medical Admission Practice Paper Generator

| | |
|---|---|
| **Status** | Draft v1 — approved for build of M0–M3 |
| **Date** | 2026-11-06 |
| **Owner** | Teacher (sole user) |
| **Stack** | Next.js (frontend) · Express.js (backend) · SQLite · Google Gemini (free tier) |

---

## 1. Background & problem

A teacher prepares a student for the **Bangladesh Medical & Dental Admission Test, 2026–27 session**. The source material is **photos** of book pages (e.g. *রেটিনা ডাইজেস্ট – সাধারণ জ্ঞান: বাংলাদেশ*):

1. **Topic analysis sheets** ("বিগত বছরসমূহের প্রশ্ন অ্যানালাইসিস") — a table of topics × sessions (২৫-২৬ … ১৫-১৬); each cell is `মে. n` (Medical), `ডে. n` (Dental), both, or `–`.
2. **MCQ pages** — Bangla-numbered questions, options ক/খ/গ/ঘ, a bracketed reference (`[DU: 25-26]`, `[MBBS: 2021-22; 36th BCS]`, `[RU (A1): 24-25]`), a printed `উত্তর: খ` box, sometimes a `ব্যাখ্যা`, and a "বিগত বছরের প্রশ্নসমূহ" section of real MBBS/BDS questions. Some questions are in English or mixed.

**Problems today**
- General AI chat tools produce PDFs with **broken Bangla** (wrong কার order, broken যুক্তাক্ষর, □ boxes) and sometimes alter text.
- Photos are angled, shadowed, curved, and carry the **student's pen ticks** over options — easy for software to misread.
- Choosing *which* questions matter most is manual; the frequency data in the topic sheets isn't used systematically.

## 2. Goals

| # | Goal | Measure |
|---|---|---|
| G1 | Turn book photos into an accurate, verified question bank | 100 % of bank questions teacher-approved; 0 invented questions/references |
| G2 | Turn topic-sheet photos into structured frequency data (Medical **and** Dental) | Grid matches photo after review |
| G3 | Rank and select the most important questions automatically | Paper of N questions built in < 1 min from an approved bank, with a reason per question |
| G4 | Be honest about uncertainty | Every unclear / doubtful item excluded and listed with its page |
| G5 | Cost nothing to run | Only Gemini free tier; runs locally |

## 3. Non-goals (this release)
- **PDF generation** (question paper & answer script) — Phase 2 (M4).
- **Student result sheet** — Phase 2 (M5).
- Multi-user accounts, login, online hosting.
- Paid AI services.
- AI-written questions of any kind.

## 4. User
One teacher, on their own Windows PC, Bangla/English bilingual, not a developer. Needs simple screens, clear status, and the photo always visible next to anything the app extracted.

## 5. Core principles (from the master prompt)
1. **Only questions from the uploaded images.** Never invent, import, or reword.
2. **Never change** words, numbers, units, options, option order, or references.
3. **Unclear → excluded and reported**, never guessed.
4. **Printed answer doubted → excluded and reported** with reasoning.
5. **Accuracy > quantity.** If fewer good questions exist than requested, deliver the maximum and say so.
6. **Honest reporting.** The app records whether each check (e.g. web search) actually ran.
7. **AI reads and suggests; code decides; teacher approves.** Scoring, numbering, selection and arithmetic are deterministic code.
8. These are **predicted practice questions**, never "guaranteed" exam questions.

## 6. User flows

### F1 — Topic Sheet portal
1. Choose subject → upload one or more sheet photos.
2. App extracts the table (two independent AI passes, compared cell by cell).
3. **Review screen**: zoomable photo left, editable grid right; disagreeing cells highlighted yellow.
4. Teacher fixes/approves → topic scores computed and shown as a ranked list.

### F2 — Question Image portal
1. Choose subject → book → drag-drop many MCQ page photos ("Image N received").
2. App extracts every question (two passes), detects the chapter header, flags cut-off / pen-obscured / unclear items.
3. **Review screen**: question crop from the photo beside editable fields; char-level diff where passes disagreed. Shortcuts: A approve, R reject, U unclear, J/K next/prev; bulk-approve items both passes agreed on.
4. Topic auto-assigned (chapter → topic match, AI classification when ambiguous); teacher can change via dropdown.
5. Duplicates grouped; references merged.

### F3 — Research
On demand per subject: Gemini with Google Search grounding summarises current syllabus, exam pattern and trends, with source links. Mapped onto the subject's topic list as a small ranking adjustment. Used **only for ranking**.

### F4 — Paper Builder
1. Settings: subject, number of questions, time, marks per question, negative marking (optional; blank fields omitted everywhere).
2. App selects from **approved** questions, balances topics, explains each choice.
3. Optional **answer verification**: AI answers each question without seeing the printed answer; mismatches are flagged and excluded by default — teacher decides.
4. Preview with swap / remove / reorder, plus the **final report** panel.

## 7. Functional requirements

### 7.1 Subjects
- FR-1 Create/edit subjects (name, language bn/en/mixed).
- FR-2 Per-subject scoring settings: Medical weight, Dental weight (default **1 : 1**), recency decay, topic cap.

### 7.2 Topic sheets
- FR-3 Upload ≥ 1 image per sheet; images normalised (EXIF rotate, resize ≤ 2400 px, original kept).
- FR-4 Extract `sessions[]` and `rows[{topic, cells[{session, medical, dental}]}]`; Bangla digits converted in code.
- FR-5 Two independent extraction passes; any cell disagreement flagged.
- FR-6 Grid editor with keyboard navigation; approve saves topics and counts.
- FR-7 Topic score:
  `score(topic) = Σ_session recency(session) × (wMed·medical + wDen·dental)`
  default recency linear from 1.0 (latest session) to 0.5 (oldest).

### 7.3 Question extraction
- FR-8 Batch upload; identical photos de-duplicated by SHA-256.
- FR-9 Extract per question: printed number, stem, 4 options, printed answer letter, reference (verbatim), explanation (verbatim, if printed), language, bounding box, chapter header.
- FR-10 Prompt rules: copy verbatim; **ignore handwriting/pen marks**; answer **only** from the printed `উত্তর:` box; never correct or reorder.
- FR-11 Flags: `cutOff`, `obscuredByPen`, `unclear` + reason → status `unclear`, excluded, listed with page.
- FR-12 Two-pass comparison after Unicode NFC + whitespace normalisation: identical → `auto_ok`; differ → `needs_review` with diff.
- FR-13 Statuses: `auto_ok · needs_review · approved · rejected · unclear`. Only `approved` can enter a paper.
- FR-14 Reference parser: `[MBBS: 2021-22; 36th BCS]` → tokens `{exam, unit, session}`; `isPastMedical` when exam ∈ MBBS / BDS / Medical / Dental. Raw string always preserved.
- FR-15 Duplicate detection: normalised stem + options trigram similarity ≥ 0.9 → one group, references merged.
- FR-16 Topic mapping: chapter header fuzzy-matched to topic names/aliases; AI classification (enum of subject topics) when ambiguous; manual override.

### 7.4 Research
- FR-17 Grounded search call → summary + sources stored with `searchRan` true/false.
- FR-18 Non-search call maps findings to topics → bounded adjustment (`inSyllabus`, `trendScore`, reason).
- FR-19 If search quota is exhausted, record that and rank from the sheet only.

### 7.5 Paper builder
- FR-20 Question score:
  `qScore = 0.60·normTopicScore + 0.25·pastMedicalBoost + 0.10·referenceRecency + 0.05·researchAdj`
- FR-21 Topic quotas proportional to topic score; no topic > 25 % of N (configurable).
- FR-22 Shortfall → deliver max available and report it.
- FR-23 One-line reason per question, generated from the score components.
- FR-24 Optional AI answer verification; mismatch → flagged, excluded by default.
- FR-25 Preview: Bangla numbering (১, ২, ৩) and ক/খ/গ/ঘ for Bangla papers; 1, 2, 3 and A/B/C/D for English papers.
- FR-26 Final report: images analysed, questions found, duplicates removed, unclear list (with page), excluded-for-answer list (with reason), requested vs delivered, subject & language, which checks ran, research done or not, and the "predicted, not guaranteed" reminder.

## 8. Non-functional requirements
- **Bangla correctness**: UI uses Noto Sans Bengali; all text stored as NFC Unicode. (PDF phase will use headless Chromium shaping with the embedded font.)
- **Security**: Gemini key only in `apps/api/.env` (gitignored); never sent to the browser; API bound to `127.0.0.1`; CORS only `http://localhost:3000`.
- **Free-tier friendliness**: rate limiting + backoff on 429/503; results cached by (image hash + prompt version + pass) so re-runs are free; long batches queued.
- **Resilience**: jobs persisted; resume after restart; progress streamed via SSE.
- **Auditability**: every AI call logged (prompt version, image hash, raw response).
- **Privacy note**: Gemini free tier may use submitted data to improve Google products — acceptable for book content.

## 9. Architecture

```
apps/web         Next.js 15 (App Router, TypeScript, Tailwind, shadcn/ui, TanStack Query)   :3000
apps/api         Express 5 (TypeScript), Prisma + SQLite, sharp, @google/genai              :4000
packages/shared  zod schemas, types, Bangla utils, scoring, selection, dedupe, diff
data/            uploads/ (originals by SHA-256), derived/ (normalised, thumbnails), app.db   [gitignored]
```

- **Gemini wrapper** `apps/api/src/ai/gemini.ts`: concurrency limit, rate limiter, retries, structured JSON output (`responseSchema`), zod validation, call logging, caching. Model from `GEMINI_MODEL` (default `gemini-2.5-flash`), verified at startup.
- **Prompts** versioned in `apps/api/src/ai/prompts/` (`topicSheet`, `mcqPage`, `classify`, `research`, `verify`).
- **Jobs** `apps/api/src/jobs/{queue,worker}.ts`: in-process queue backed by the `Job` table.

### Data model (Prisma)
- `Subject` (name, language, scoring settings)
- `TopicSheet` → `Topic` (name, order, chapterAliases) → `TopicCount` (session, medical, dental)
- `Book` → `PageImage` (sha256, path, chapterHeader, status)
- `Question` (printedNumber, stem, optA–D, printedAnswer, referenceRaw, referenceTokens, explanationRaw, language, bbox, topicId, status, unclearReason, isPastMedical, duplicateOfId)
- `ResearchNote` (summary, sources, searchRan) · `TopicAdjustment` (inSyllabus, trendScore, reason)
- `Paper` (settings) → `PaperItem` (order, questionId, reason, verifyStatus)
- `Job`, `AiCall`

### Key files
- `apps/api/src/server.ts`, `routes/{subjects,topicSheets,pages,questions,research,papers,jobs}.ts`
- `apps/api/src/images/normalize.ts`
- `packages/shared/src/{schemas,bangla,reference,scoring,selection,dedupe,diff}.ts`
- `apps/web/app/(portal)/{topic-sheets,images,bank,research,papers}/…`
- `apps/web/components/{ImageViewer,SheetGridEditor,QuestionReviewCard,JobProgress}.tsx`

## 10. Milestones

| Milestone | Scope | Done when |
|---|---|---|
| **M0 Scaffold** | Monorepo, TS, lint, Prisma/SQLite, `/api/health`, `/api/health/gemini`, Next.js shell with sidebar (Subjects · Topic Sheets · Question Images · Question Bank · Research · Papers) | Both apps run with `npm run dev`; Gemini health check passes |
| **M1 Topic Sheets** | F1, FR-1…FR-7 | Sample sheet extracted, reviewed, scores shown |
| **M2 Question Images** | F2, FR-8…FR-16 | 5 sample pages extracted; pen ticks ignored; edge questions marked unclear; review flow works |
| **M3 Research + Bank + Papers** | F3, F4, FR-17…FR-26 | 20-question paper built with reasons, topic balance and final report |
| M4 PDFs *(later)* | Puppeteer + embedded Noto Bengali; QC: text extraction vs DB, page renders, student-paper leak check | — |
| M5 Result sheet *(later)* | Answer grid, deterministic scoring, optional negative marking, topic-wise weakness | — |

## 11. Acceptance & testing
- **Unit (vitest, `packages/shared`)**: Bangla digits, NFC normalise/diff, reference parser on every format seen (`[DU: 25-26]`, `[MBBS: 2021-22; 36th BCS]`, `[RU (A1): 24-25]`, `[44th, 38th BCS, DU (B): 14-15]`, `[JnU (B): 16-17]`, `[GST: 25-26]`), scoring, quota selection, dedupe.
- **API integration (vitest + supertest)** with Gemini mocked from recorded fixtures.
- **Live end-to-end** with the sample photos: topic grid matches the photo; ≥ 20 questions spot-checked character by character; printed `উত্তর` used, not pen ticks; cut-off questions excluded; paper of 20 built with reasons.
- **Browser check** of Bangla rendering and review flows.

## 12. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Gemini misreads Bangla (e.g. ১ vs ৯, কার signs) | Two-pass diff + mandatory teacher review beside the photo |
| Pen ticks mistaken for answers | Prompt rule + answer only from printed box + `obscuredByPen` flag |
| Angled / shadowed photos | Image normalisation; guidance to shoot flat, from above, in good light; unclear → excluded |
| Free-tier rate / daily limits change | Configurable limiter, caching, queued jobs, model name via env |
| Search quota exhausted | Recorded honestly; ranking falls back to topic sheet |
| Model names change over time | `GEMINI_MODEL` env + startup verification |

## 13. Open questions
1. Subjects beyond সাধারণ জ্ঞান (Physics, Chemistry, Biology, English) — each needs its own topic sheet.
2. Answer script (Phase 2): printed ব্যাখ্যা only, or also AI-written explanations clearly marked as AI-written?

## 14. Action items for the user
- **Delete the Gemini API key shown in the earlier screenshot and create a new one**; paste it yourself into `apps/api/.env` once M0 is set up.
- Photograph pages flat, from directly above, in good light.
