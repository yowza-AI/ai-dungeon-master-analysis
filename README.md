# 🐉 AI Dungeon Master: Tool-Calling D&D 5e Rules Engine + Tactics AI

> **The problem**: Running D&D requires a Dungeon Master (DM) - the person running the game for the rest of the group.  Experienced DMs are under-represented in the D&D community, and many groups of friends are left wanting to play the game, but unable due to the lack of DMs.  Even when one is found, the Dungeon Master must memorize combat rules, manage initiative, calculate damage, track conditions, and optimize NPC tactics—all while narrating. It's cognitive overload.

**AI Dungeon Master solves this** with a deterministic rules engine that handles all D&D 5e mechanics, a tactics engine that plans NPC combat strategy, and LLM narration that brings it to life. The DM narrates; the AI handles the math.

---

## What It Does

### Deterministic Rules Engine (D&D 5e SRD)

- **Attack resolution**: bonuses, advantage/disadvantage, critical hits, damage rolls
- **Saving throws & ability checks**: DC scaling, proficiency, custom modifiers
- **Spells & abilities**: parsed from SRD, resolved with correct mechanics
- **Conditions**: tracking effects (grappled, restrained, poisoned, etc.)
- **Opportunity attacks**: triggered automatically when enemies leave melee reach
- **Action economy**: bonus actions, reactions, movement per turn

### Combat Tactics Engine (Deterministic, Intelligent)

Creatures fight smart without an LLM per turn:
- **Target scoring**: prioritize low-HP enemies, casters, threats
- **Movement**: calculate reachable targets and optimal positioning
- **Action economy**: choose attack/dash/disengage/dodge based on situation
- **Intelligence dial**: MINDLESS (attacks nearest), ANIMAL (weak self-preservation), SMART (focus-fire, kiting, retreat)
- **Opportunity attacks**: enemies get reactions when creatures leave melee range

### DM Agent (Tool-Calling Loop)

LLM-powered adjudication:
- Player describes action in free form
- DM agent parses intent and decides which mechanic applies
- Calls deterministic rules tools (never invents numbers)
- Receives actual dice results
- Narrates what happens, incorporating real outcomes

### Character Persistence

- **NPC stat blocks**: stored, reused across sessions
- **Combatants**: HP, conditions, position tracked during encounter
- **Voice/personality**: set per NPC for flavor narration during turns

---

## Architecture

Player Action (free form)
  ↓
DM Agent (LLM + tools)
  ├─ Parse intent
  ├─ Decide which mechanic
  └─ Call appropriate tool(s)
       ↓
Rules Engine (deterministic)
  ├─ Attack rolls
  ├─ Damage rolls
  ├─ Saving throws
  ├─ Condition tracking
  └─ Opportunity attacks
       ↓
Tool Results (actual numbers)
  ↓
DM Agent (narration)
  └─ Incorporates real outcomes
       ↓
NPC Turn (if combat)
  ↓
Tactics Engine (deterministic)
  └─ Plan optimal action
       ↓
NPC Agent (executes + narrates)
  ├─ Movement
  ├─ Attacks (with opportunity AoO)
  ├─ Conditions
  └─ Flavor narration

---

## Key Architectural Decisions

### 1. Mechanics as Deterministic Tools (Not LLM Output)

Every number comes from tools: Initiative rolls, Attack bonuses, Damage dice, Saving throws, Condition applications.

The LLM **never** invents mechanics. It decides *which* tool to call, then narrates the actual result.

### 2. Tactics as Pure Heuristics (Not LLM per Turn)

Combat planning is **deterministic**:
- Creature intelligence dial (MINDLESS / ANIMAL / SMART)
- Target scoring based on threat
- Movement to optimal position
- Fast (~milliseconds per turn), repeatable, no LLM tokens

An optional LLM adds 1-2 sentences of flavor *after* mechanics are locked in (TIER_EASY, cheap).

### 3. Tool-Calling DM Agent (Free-Form Player Input)

Players say what they *do*, not the mechanics:
- "I try to grapple the orc" → agent calls make_contested_check (STR vs STR)
- "I cast fireball on the cluster" → agent calls cast_spell(fireball, targets)
- "I disengage and dash to the door" → agent calls movement then dash

The agent parses intent and **calls the right tools**; it never computes anything.

### 4. NPC Turns: Deterministic Plan → Tool Execution → Optional Flavor

Tactical Plan (from heuristics) → NPC Agent executes via tools → Tool results locked in → Optional LLM flavor (in-character, doesn't change mechanics)

### 5. Character Persistence (Database)

NPCs are database records:
- Stat blocks (AC, HP, attacks, spells, abilities)
- Personality / voice description (for flavor)
- Combat state during encounter (HP, conditions, position)
- Reused across sessions

### 6. LLM Provider Abstraction

Works with either:
- **Claude** (via Anthropic API) — for complex narration
- **Local** (Ollama) — for privacy, no API keys

Same session logic, different backend.

---

## Tech Stack

| Component | Technology | Why |
|-----------|-----------|-----|
| **Rules Engine** | Python + custom logic | Deterministic, fast, SRD-compliant |
| **Combat AI** | Pure heuristics (no LLM) | Fast, repeatable, tunable intelligence dial |
| **DM Agent** | LLM + tool calling | Parses free-form input, calls mechanics tools |
| **LLM Backend** | Claude or Ollama | Narration, flavor, adjudication |
| **Data** | SQLite | NPC stat blocks, campaign state, combat log |

---

## Development Approach


This demonstrates: building a **domain-specific tool system** where mechanics are deterministic (never ambiguous), tactics are algorithmic (fast + repeatable), and LLM narration enhances without changing outcomes. The AI handles the burden; the DM keeps control.

---

**Status**: MVP with core mechanics working  
**Philosophy**: Tools handle mechanics, LLM handles narration  
**Core Loop**: Player action → DM agent (tools) → Rules engine → Narration
