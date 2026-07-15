# AI Storyteller: Architectural Decision Records

A multi-agent personalized bedtime story generator with local TTS, consistent character voices, and emotion-aware narration.

---

## ADR-1: Multi-Agent Writers' Room (Not Single-Agent Generation)

**Decision**: Story generation is a pipeline: Writer → QA → Manager → Casting, not a single LLM call.

**Why This Matters**: Single-agent generation produces decent stories but inconsistent quality. Multi-agent pipelines catch errors and enforce guardrails.

**Pipeline**:
```
Writer Agent (drafts the story)
  ↓ (creative, 1500-word target)
QA/QC Agent (checks age-rating + flow)
  ↓ (1-2 revision rounds)
Manager Agent (final approval + quality gate)
  ↓ (acts as editor-in-chief)
Casting Agent (detects characters, designs voices)
  └─→ Character library updated
```

**Why This Choice**:
- **Writer** is creative but unsupervised → stories can slip past age-rating ceiling
- **QA** catches violations → revises or rejects before manager sees it
- **Manager** makes final call → enforces quality bar (coherence, ending quality, pacing)
- **Casting** ensures character consistency → same character, same voice, every story

**Trade-off Accepted**: 2-3 LLM passes per story instead of 1. Worth it because output quality improves significantly.

**Metrics**:
- Age-rating compliance: >95% pass on first QA pass
- Revision rate: 40-60% of stories get 1-2 rounds of QA refinement
- Manager rejection rate: <5% (most make it through)

---

## ADR-2: Emotion-Aware Narration (Per-Line Emotion Metadata)

**Decision**: Each story line carries emotion metadata. TTS respects it.

**Why It Matters**: Stories need to *feel* alive. Flat narration is boring; per-line emotion makes stories engaging.

**Pipeline**:
1. **Script generation**: Writer produces story + Casting agent annotates each line with emotion
2. **Emotion metadata**: scared, excited, sleepy, thoughtful, mischievous, etc.
3. **TTS rendering**: VoiceDesign model reads emotion description, generates voice to match
4. **Result**: Characters sound scared when they're scared, excited when they're excited

**The Voice Strategy**:

| Voice Type | Model | Consistency | Emotion | Use Case |
|-----------|-------|-------------|---------|----------|
| **Narrator** | Base (voice clone) | ✅ High (reproducible) | ❌ None | Steady, gravelly narration |
| **Characters** | VoiceDesign (emotion TTS) | ⚠️ Medium (slight drift) | ✅✅ Rich | Expressive, emotion-driven |

**Why This Choice**:
- **Narrator = consistent + steady**: Kids recognize the storyteller's voice across all stories
- **Characters = expressive + unique**: Each emotion carries distinct voice quality
- **Trade-off accepted**: VoiceDesign has slight voice drift between lines (emotion > consistency for characters)

**Example**:
- Narrator (calm, gravelly): "The dragon stirred in its cave."
- Young girl (terrified, high, breathless): "P-please don't wake him up!"
- Wise wizard (thoughtful, deep, measured): "Perhaps... we should wait until dawn."

---

## ADR-3: Character Persistence (Database-Backed Voice Mapping)

**Decision**: Characters are persistent entities. First mention → design + voice mapping. Reuse in all future stories.

**Why It Matters**: Character continuity across stories creates emotional attachment. Kids want to hear from their favorite characters again.

**Data Model**:
```
Character
├─ persona (age, role, personality)
├─ backstory (history, motivations)
├─ voice_mode (preset | clone)
├─ speaker (preset name like "Aiden" or cloned reference)
├─ voice_instruct (emotion description or voice style)
└─ ref_audio_path, ref_text (for voice cloning)
```

**Casting Agent Workflow**:
1. Story is written, characters are mentioned
2. Casting agent scans for *new* characters (not in database)
3. For each new character:
   - Design persona (age, role, personality, motivations)
   - Write voice description (e.g., "A young girl about ten, voice high and trembling, breathless and terrified")
   - Map to voice: preset (Aiden, Ryan) or custom (VoiceDesign description)
   - Persist to database
4. In future stories, same character uses same voice

**Why This Choice**:
- **First mention**: Casting agent designs character once
- **Reuse**: Every subsequent mention uses the same voice
- **Emotional attachment**: Kids recognize voices → emotional investment in characters
- **Efficiency**: Don't re-design the same character in every story

**Trade-off Accepted**: Casting adds a 4th agent to the pipeline. Worth it for character continuity.

---

## ADR-4: Local-First Architecture (Privacy by Default)

**Decision**: All generation on the user's machine. No cloud APIs, no story collection.

**Why It Matters**: Parents care about privacy. Stories about their kids should never leave their hard drive.

**Architecture**:
- **Ollama**: Story generation runs locally (qwen2.5:7b-instruct)
- **Qwen3-TTS**: Voice generation runs locally (1.7B model, FP16 on GPU)
- **SQLite**: Stories stored in local database (gitignored, never synced)
- **No API calls**: Zero external dependencies once models cached

**Why This Choice**:
- **Privacy**: Parent owns the data. No surveillance, no analytics, no data sharing.
- **Offline capable**: Works without internet once models are downloaded
- **Fast iteration**: No API rate limits, no latency from cloud calls
- **Cost model**: One-time setup; zero marginal cost per story

**Constraints Accepted**:
- GPU memory limited (local card vs. cloud GPUs)
- Setup complexity (download large models, GPU drivers)
- Scaling (vertical only — one machine per user)

**Fallbacks**:
- TTS disabled? App still works with silent script
- Emotion TTS unavailable? Fall back to preset voices
- Ollama down? App returns error (user restarts)

---

## ADR-5: Qwen3-TTS with VoiceDesign (Emotion Over Consistency)

**Decision**: Use VoiceDesign (emotion-aware) for character voices, Base (voice clone) for narrator.

**Why It Matters**: Character expressiveness is more important than per-line consistency. Emotion drives engagement.

**Trade-off Analysis**:

| Approach | Emotion | Consistency | Complexity | Result |
|----------|---------|-------------|-----------|--------|
| **Preset only** | ❌ Weak (fixed voices) | ✅ High | Low | Flat, boring |
| **VoiceDesign only** | ✅✅ Rich | ⚠️ Medium | High | Expressive, slight drift |
| **Hybrid (chosen)** | ✅✅ Rich (chars) + ✅ Steady (narrator) | ✅ High | Medium | Best of both |

**Hybrid Approach**:
- **Narrator (Base model)**: Voice clone of a reference clip (gravelly British voice)
  - Reproducible across all stories
  - Steady, consistent tone
  - No per-line emotion (clone preserves reference delivery)
- **Characters (VoiceDesign model)**: Per-line emotion description
  - "A young girl about ten, her voice high and trembling, breathless and terrified"
  - Emotion-driven, expressive
  - Slight voice drift between lines (acceptable trade-off)

**Rendering Sequence**:
1. **Pass A**: Load Base model, render all narrator clips (slow, ~3–4× preset speed)
2. **Pass B**: Unload Base, load VoiceDesign, render all character clips
3. **Concatenate** with smooth gaps (~0.28s) in story order
4. Typical time: ~30 min for ~10 min story on 2080-series GPU

**Why This Choice**:
- A/B testing showed users strongly prefer **VoiceDesign for emotion** over preset tweaking
- Narrator consistency matters for brand recognition (same storyteller every night)
- Character expressiveness matters for engagement (kids react to emotion in voice)

---

## ADR-6: Graceful Fallbacks (TTS Optional, Story First)

**Decision**: TTS is a feature, not a requirement. App works without it (silent script mode).

**Why It Matters**: Users might not have GPU, or Qwen3-TTS setup might fail. Story generation should still work.

**Fallbacks**:
1. **TTS fully working**: Full narration with emotion-aware voices
2. **Emotion TTS unavailable**: Fall back to preset voice + weak instruct
3. **All TTS unavailable**: Generate silent script (text + character + emotion, no audio)
4. **No Ollama available**: Return error (user restarts)

**Why This Choice**:
- **TTS is complex**: Model downloads, CUDA setup, version conflicts
- **Story generation is core**: If TTS fails, don't throw away the story
- **User experience**: Parent gets the story script to read aloud manually
- **Graceful degradation**: Partial features > total failure

**Implementation**:
- All TTS calls wrapped in try/except
- If TTS fails, story.audio_path = None (UI shows "Silent script available")
- Parent can still read the story to their child

---

## ADR-7: SQLite + Ollama Config (Simplicity Over Scaling)

**Decision**: SQLite for stories, Ollama for writing. No cloud database, no managed services.

**Why It Matters**: Simplicity. One machine, no ops burden, no syncing headaches.

**Schema**:
- subjects: child profiles
- characters: persistent character library
- stories: story metadata + full text
- story_lines: playwright script (per-line character + emotion + text + audio path)

**Why This Choice**:
- **Zero operations**: SQLite is a file, no database server
- **Portable**: Backup is just cp data.db backup.db
- **Good enough**: Single user per machine; no concurrency issues
- **Full ACID**: Transactions work, no eventual consistency headaches

**Trade-off Accepted**: Doesn't scale to 1000s of concurrent users. Accept that limitation.

---

## Summary: Core Principles

| Principle | Architectural Response |
|-----------|------------------------|
| **Quality over speed** | Multi-agent pipeline (Writer → QA → Manager) |
| **Emotion matters** | Per-line emotion metadata + VoiceDesign TTS |
| **Character continuity** | Persistent character database + voice mapping |
| **Privacy first** | All local, no cloud, no data collection |
| **Graceful degradation** | TTS optional; story works without it |

---

**Document Owner**: Architecture team  
**Last Updated**: July 2026
