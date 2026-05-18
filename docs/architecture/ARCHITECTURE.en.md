# Second Brain Ecosystem — Architecture

## The Story

I needed to solve a concrete problem: **managing all the information on my PC**.

I started building a virtual library for everything that caught my attention — technical knowledge, books, podcasts, YouTube video summaries. I wanted my own curated, organized knowledge library.

It all started with the [LLM Wiki gist by Karpathy](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f). I decided to build a second brain. At first, it was just two folders with hand-written `.md` files. It worked, but eventually managing them became complicated: broken links, duplicate notes, stale content, orphaned ideas.

So I decided to **automate my Second Brain**.

The ecosystem now has these components:

- **Obsidian** — The vault. Where all knowledge lives. The human interface.
- **Librarian** — A review-driven knowledge maintenance pipeline for the Obsidian vault. It reads sources explicitly placed in `raw/`, classifies, detects duplicates, generates proposals, and applies approved changes to the wiki. All mutations require human approval.
- **Content Toolkit** — Ingestion tools. Video/audio transcription with Whisper, full YouTube → audio → transcription → smart summary pipeline. It produces artifacts that I can explicitly save into `raw/` when I want Librarian to process them.
- **Researcher** — A web search agent (Search-o1 pattern: think → search → read → synthesize). It finds information missing from the library and fills in the blanks. It doesn't depend on Librarian — it's invoked directly, and its output can be saved into `raw/` when useful.

It's in alpha. It may have bugs. But it's already useful.

---

## Architecture Diagram

```mermaid
graph TB
    subgraph USER["👤 User"]
        U1["Capture ideas, notes, links"]
        U2["Chat with Librarian<br/>via TUI / CLI"]
        U3["Approve / Reject<br/>proposals"]
        U5["Explicit consent<br/>move/copy into raw/"]
        U4["Read & navigate<br/>wiki in Obsidian"]
    end

    subgraph SOURCES["📥 External Sources"]
        YT["📺 YouTube"]
        POD["🎙️ Podcasts"]
        WEB["🌐 Web / Papers"]
        PDF["📄 PDFs / Articles"]
        MANUAL["📝 Manual notes"]
    end

    subgraph TOOLKIT["🔧 Content Toolkit"]
        INGEST["ingest-youtube<br/>yt-dlp → ffmpeg → Whisper"]
        TRANSCRIBE["transcribe<br/>Whisper standalone"]
        ARTIFACT["transcripts / summaries<br/>staged for review"]
    end

    subgraph VAULT["🗄️ Obsidian Vault"]
        INBOX["inbox/<br/>Human capture<br/>(not read by Librarian)"]
        RAW["raw/<br/>Immutable sources<br/>approved for AI processing"]
        WIKI["wiki/<br/>Curated knowledge<br/>concepts · entities<br/>sources · synthesis"]
        REVIEWS["reviews/<br/>Human-readable<br/>proposal review surface"]
        PROPOSALS[".librarian/proposals/<br/>Internal proposal<br/>source of truth"]
        LEDGER[".librarian/processed.json<br/>Processed source ledger"]
    end

    subgraph LIBRARIAN_AGENT["📚 Librarian (Curator Agent)"]
        L1["Inspect raw/"]
        L2["Classify & Dedup"]
        L3["Generate Proposals"]
        L4["Apply<br/>(with human approval)"]
        L1 --> L2 --> L3
    end

    subgraph RESEARCHER_AGENT["🔍 Researcher (Search Agent)"]
        R1["THINK"]
        R2["SEARCH<br/>Brave / SearXNG / Tavily"]
        R3["READ & Synthesize"]
        R1 --> R2 --> R3
    end

    %% Sources → Content Toolkit → staged artifacts
    YT --> INGEST
    POD --> TRANSCRIBE
    INGEST -->|"transcription + summary"| ARTIFACT
    TRANSCRIBE -->|"transcription"| ARTIFACT
    ARTIFACT -->|"user reviews output"| U5

    %% Manual sources → inbox/ or raw/ via explicit consent
    WEB -->|"manual capture"| INBOX
    PDF -->|"manual capture"| INBOX
    MANUAL --> INBOX
    U1 --> INBOX
    U1 -->|"direct approved source"| U5
    INBOX -->|"promote intentionally"| U5
    U5 --> RAW

    %% raw/ → Librarian → proposals → approval → apply → wiki/
    RAW --> L1
    L3 -->|"stores proposal"| PROPOSALS
    L3 -->|"exports readable review"| REVIEWS
    REVIEWS -.->|"pending proposals"| U3
    PROPOSALS -->|"approved proposal"| L4
    L4 -->|"writes curated notes"| WIKI
    L4 -->|"marks source processed<br/>after successful apply"| LEDGER

    %% User interacts with Librarian
    U2 --> L1
    U3 -->|"approve / reject via CLI"| PROPOSALS
    U3 -->|"apply approved proposal"| L4

    %% Researcher → can feed raw/ only through user consent
    R3 -->|"answer + sources"| U5

    %% User reads wiki
    WIKI --> U4

    %% Researcher invoked directly
    U2 -.->|"or"| R1

    %% Styling
    classDef user fill:#f9e79f,stroke:#f4d03f,color:#000
    classDef source fill:#aed6f1,stroke:#5dade2,color:#000
    classDef toolkit fill:#d5f5e3,stroke:#58d68d,color:#000
    classDef vault fill:#fadbd8,stroke:#ec7063,color:#000
    classDef agent fill:#e8daef,stroke:#af7ac5,color:#000
    classDef research fill:#fdebd0,stroke:#f0b27a,color:#000

    class U1,U2,U3,U4,U5 user
    class YT,POD,WEB,PDF,MANUAL source
    class INGEST,TRANSCRIBE,ARTIFACT toolkit
    class RAW,WIKI,INBOX,REVIEWS,PROPOSALS,LEDGER vault
    class L1,L2,L3,L4 agent
    class R1,R2,R3 research
```

---

## Communication Flow

### 1. User ↔ Obsidian

The user interacts directly with Obsidian as the main interface. Writes notes, captures ideas, reviews generated wiki pages, and navigates the knowledge graph. Everything is plain Markdown.

```
User → Obsidian (vault/)
User ← Obsidian (read wiki/, search, navigate)
```

### 2. User ↔ Librarian

The user communicates with Librarian through a TUI (terminal) or CLI. Can ask it to ingest approved sources, search the wiki, or run maintenance. Librarian generates proposals before any mutation.

`reviews/` is a human-readable review and export surface. `.librarian/proposals/` is the internal proposal source of truth.

```
User → Librarian TUI/CLI → "ingest approved sources", "search wiki for X"
Librarian → .librarian/proposals/ → stores proposal
Librarian → reviews/ → exports readable proposal
User → Librarian → "approve proposal #42"
User → Librarian → "apply proposal #42"
Librarian → wiki/ → writes curated note
Librarian → .librarian/processed.json → marks source processed
```

### 3. Content Toolkit → User → Vault (raw/)

Content Toolkit is a pre-processor. It transforms media (video, audio) into text, but it does not decide what Librarian can process. The user reviews the output and explicitly saves or moves it into `raw/` when it should enter the AI-processing layer.

```
YouTube URL → ingest-youtube → transcription + summary → user review → raw/
Video/Audio → transcribe → transcription → user review → raw/
```

### 4. Researcher (independent)

Researcher is a web search agent. It doesn't depend on Librarian — it's invoked directly. It searches the web, reads pages, and synthesizes answers. Its output can be copied to `raw/` by the user for Librarian to process later.

```
User → researcher "what is agentic RAG?" → answer + sources
User reviews result → raw/ → (optional) Librarian processes it
```

---

## Ecosystem Components

| Component | Role | Repo |
|-----------|------|------|
| **Obsidian** | Human interface, knowledge vault | Local vault |
| **Librarian** | Review-driven, proposal-based knowledge maintenance pipeline | [`librarian`](https://github.com/Agents4Life/librarian) |
| **Content Toolkit** | Media ingestion (YouTube → text, transcription) with user-approved vault entry | [`content-toolkit`](https://github.com/VanessaPellegrini/content-toolkit) |
| **Researcher** | Agentic web search (Search-o1) | [`researcher`](https://github.com/Agents4Life/researcher) |

---

## Key Decisions

- **Everything Librarian processes enters through `raw/` first** — `raw/` is the explicit consent boundary for AI processing.
- **Librarian doesn't search the web** — Its scope is Obsidian management, not Google.
- **Researcher is a separate repo** — Single responsibility, reusable.
- **Content Toolkit is a pre-processor** — Transforms media before the user decides whether the result belongs in `raw/`.
- **Proposals before apply** — Never write directly to `wiki/` without human approval and an explicit apply step.
- **Reviews and proposals are separate surfaces** — `reviews/` is for human review; `.librarian/proposals/` is the internal source of truth.
- **Wikilinks > tags** — The connection graph is more valuable than categories.
