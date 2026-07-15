# 🎭 AI Storyteller: Personalized Bedtime Stories with Consistent Character Voices

> **The problem**: Mass-produced bedtime stories are generic. They don't fit your child's interests, age, or attention span. Characters change voices between stories. Parents read the same dozen books on rotation.

**AI Storyteller solves this** with a personalized story generation engine: write unique stories tailored to your child's preferences, with consistent character voices, proper age-rating, and emotion-aware narration — all generated locally on your machine.

---

## What You Can Do

### Generate Stories Personalized to Your Child

1. **Enter your child's name, age, interests**
2. **Pick a genre** (Fluffy, Fantasy, Sci-Fi, Suspense, Horror)
3. **Set a content ceiling** (G, PG, PG-13, R — "Horror+G" is charming-spooky; "Horror+PG-13" is genuinely eerie)
4. **Choose a cast** (reusable characters with persistent voices)
5. **Generate** (multi-agent writers' room writes, QA refines, manager approves)
6. **Play** (local narration with emotion-aware voices)

### Character Persistence & Voice Continuity

- **First mention**: The system detects new characters and designs them (persona, motivations, backstory, voice description)
- **Maps to voice**: Each character gets a unique voice with their emotion-style baked in
- **Reused every story**: Same character = same voice across all future stories
- Result: Kids recognize their favorite characters instantly

### Emotion-Aware Narration

- **Per-line emotion control**: Each story line carries emotion metadata (sad, excited, terrified, sleepy)
- **Voice generation respects it**: Narrator and characters speak with appropriate emotion, not flat
- **Multi-voice casting**: Narrator (steady, gravelly), main characters (expressive), background characters (distinct)

### All Local—Privacy Included

- Runs on your machine (Ollama + local TTS)
- Stories never leave your hard drive
- No cloud API calls, no tracking, no data collection
- Works offline once models are cached

---

## How It Works

### The Writers' Room (Multi-Agent Pipeline)

```
📝 Writer Agent
  ↓ (writes initial story)
🔍 QA/QC Agent
  ↓ (checks age-rating, story flow)
✅ Manager Agent
  ↓ (final approval + quality gate)
🎭 Casting Agent
  └─→ Character Detection & Voice Design
      ├─ Detects new characters
      ├─ Designs persona + voice description
      ├─ Maps to voice preset or designs custom voice
      └─ Persists for reuse in future stories
```

Each story goes through 1-2 QA rounds (configurable). The manager has final say on age-rating compliance.

### Narration Pipeline

**The Voice Strategy**:
- **Narrator** (steady, gravelly clone): reads all narration lines consistently
- **Characters** (emotion-rich VoiceDesign): each line carries emotion metadata (scared, excited, thoughtful), reflected in voice

**Rendering**:
1. Build script: per-line character + emotion + text
2. **Pass A**: Render all narrator clips via voice clone (reproducible, steady)
3. **Pass B**: Render all character clips via emotion-aware VoiceDesign (expressive, unique per emotion)
4. **Concatenate** with smooth gaps (~0.28s) in original order

**Time**: ~30 min for a ~10-min story on a 2080-series GPU (generate-ahead-of-bedtime queue).

---

## Architecture

```
┌──────────────────────────────┐
│ Frontend (Vanilla HTML/JS)   │
│ ├─ Child profile             │
│ ├─ Story parameters          │
│ ├─ Character library         │
│ └─ Playback + history        │
└────────────┬─────────────────┘
             ↓
┌──────────────────────────────────┐
│ Backend (FastAPI)                │
│ ├─ Writers' room (Ollama)       │
│ │  ├─ Writer agent             │
│ │  ├─ QA/QC agent              │
│ │  ├─ Manager agent            │
│ │  └─ Casting agent            │
│ ├─ Narration pipeline           │
│ │  ├─ Voice cloning (Base TTS)  │
│ │  ├─ Emotion TTS (VoiceDesign) │
│ │  └─ Concatenation            │
│ └─ Data access (SQLite)         │
└────────────┬─────────────────────┘
             ↓
┌──────────────────────────────────┐
│ Local Models (GPU)               │
│ ├─ Ollama (story writing)        │
│ │  └─ qwen2.5:7b-instruct       │
│ └─ Qwen3-TTS (narration)        │
│    ├─ Base model (voice cloning)│
│    └─ VoiceDesign (emotion TTS) │
└──────────────────────────────────┘
```

**Key Design Decisions:**

1. **Multi-Agent Writers' Room**
   - Writer generates a rough draft
   - QA checks for age-rating compliance and story flow
   - Manager makes final quality call
   - Result: Better stories than any single agent

2. **Character Persistence**
   - Casting agent detects new characters and designs them
   - Voice is persisted in the database
   - Reused in every future story
   - Kids recognize their favorite characters instantly

3. **Emotion-Aware Narration**
   - Each line carries emotion metadata (scared, excited, sleepy)
   - VoiceDesign TTS respects emotion in voice generation
   - Narrator stays steady (clone), characters are expressive (VoiceDesign)
   - Result: Stories feel alive, not monotone

4. **Local-First Architecture**
   - All generation on your machine (Ollama + local TTS)
   - Stories are private by default
   - No cloud dependency, no tracking
   - Works offline once models cached

5. **Graceful Fallbacks**
   - TTS disabled? App still runs with silent script
   - Can use any Ollama-compatible model, not just Qwen
   - Emotion TTS unavailable? Fall back to preset voices

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| **Frontend** | HTML + Vanilla JS + CSS | Simple, no dependencies, instant UI |
| **Backend** | FastAPI + Uvicorn | Fast async API, minimal overhead |
| **Story Writing** | Ollama (qwen2.5:7b-instruct) | Open, runs locally, good narrative quality |
| **Narration** | Qwen3-TTS 1.7B (FP16 on GPU) | Emotion-aware, voice cloning, runs on 2080s |
| **Storage** | SQLite | Reliable, no server, story persistence |
| **Data** | Character library, story archive | Enables continuity and discovery |

---

## Development Approach


This demonstrates: building a **multi-agent creative system** where the architecture actively improves output quality (writers' room → QA → approval), and where persistent character state (voices) creates continuity across generated content.

---

**Status**: Working MVP  
**Users**: Parents, bedtime-time readers, voice enthusiasts  
**Philosophy**: Local-first, privacy-respecting, emotion-aware storytelling
