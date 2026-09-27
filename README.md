# Cadence

Every recording, transcribed, summarized, and ready to talk to.

Cadence is a full-stack app that turns audio recordings — meetings, interviews,
speeches, voice notes — into searchable transcripts, speaker-labeled conversations,
AI-generated summaries, and a chat interface for asking questions about what was said.

<!-- 📸 ADD SCREENSHOT: Landing page hero -->
![Landing page](docs/screenshots/landing.png)

## Features

- 🎙️ **Transcription** — powered by faster-whisper, with word-level confidence flagging
- 🗣️ **Speaker diarization** — automatically separates who said what in meetings
- 📝 **AI summaries** — key points and action items, generated automatically
- 💬 **Ask anything** — chat with your recording after it's processed, with full history saved
- 📤 **Export** — download transcripts and summaries as .txt, .md, or .pdf
- 🔒 **Per-user isolation** — Supabase Auth + Row-Level Security, every user only sees their own data

<!-- 📸 Transcript tab with speaker labels -->
![Transcript view](docs/screenshots/transcript.png)

<!-- 📸 Summary tab -->
![Summary view](docs/screenshots/summary.png)

<!-- 📸 Ask/chat tab -->
![Chat view](docs/screenshots/chat.png)

## Tech stack

**Frontend:** Vanilla HTML/CSS/JS (no build step), Supabase Auth (direct REST calls)

**Backend:** FastAPI, faster-whisper (transcription), pyannote (speaker diarization),
Gemini API (summarization), Supabase (Postgres + Storage + Auth)

## Architecture

<!-- 📸 architecture / data flow -->
```mermaid
flowchart TD
    A[Browser: Cadence frontend] -->|Supabase Auth REST| B[(Supabase Auth)]
    A -->|HTTPS via ngrok| C[FastAPI backend on Colab]
    C -->|reads/writes| D[(Supabase Postgres<br/>jobs, qa_messages)]
    C -->|upload/download| E[(Supabase Storage<br/>job-files bucket)]
    C -->|transcribe| F[faster-whisper]
    C -->|diarize| G[pyannote speaker diarization]
    C -->|summarize + format| H[Gemini API]
    C -->|answer questions| I[sentence-transformers retriever]

    subgraph Processing Pipeline
        F --> G --> H
    end
```

## Setup

### Backend

The backend runs as a Jupyter/Colab notebook (`backend/cadence_backend.ipynb`).

1. Open the notebook in Google Colab
2. Add these secrets via Colab's 🔑 secrets panel (never hardcode them):
   - `SUPABASE_URL`
   - `SUPABASE_SERVICE_ROLE_KEY`
   - `GEMINI_API_KEY` (or your chosen LLM provider key)
3. Run all cells top to bottom
4. Copy the printed `ngrok` URL — you'll need it for the frontend

### Database setup (Supabase)

Run this in your Supabase SQL Editor:

```sql
-- Core tables
create table jobs (
  id uuid primary key,
  user_id uuid not null,
  status text,
  purpose text,
  filename text,
  title text,
  transcript text,
  speaker_transcript text,
  summary text,
  transcript_url text,
  summary_url text,
  transcript_word_confidence jsonb,
  formatted_output text,
  error text,
  created_at timestamptz default now()
);

create table qa_messages (
  id uuid primary key default gen_random_uuid(),
  job_id uuid references jobs(id),
  question text,
  answer text,
  created_at timestamptz default now()
);

-- Row-Level Security
alter table jobs enable row level security;
alter table qa_messages enable row level security;
```

Create a Storage bucket named `job-files`, and add a policy allowing `service_role` full access.

### Frontend

1. Open `frontend/cadence.html` in a browser, or host it anywhere static (GitHub Pages, Netlify, etc.)
2. Click the ⚙ settings icon and fill in:
   - Your Supabase project URL
   - Your Supabase **anon** key (Settings → API in Supabase dashboard — not the service role key)
   - Your backend's `ngrok` URL
3. Sign up and start uploading recordings

## Known limitations

- Backend runs on Colab + ngrok, so the URL changes on every restart (not production-hosted yet)
- Jobs process sequentially, one at a time (single background worker thread)
- Gemini free tier caps at 20 requests/day — summarization may fail under heavy testing

## License

All rights reserved. This code is shared publicly for portfolio/demonstration purposes only — 
it is not licensed for reuse, modification, or redistribution without explicit permission.