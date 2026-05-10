# Life Knowledge Base Routing Protocol

This skill documents the protocol for navigating and querying the hierarchical Life Knowledge Base located at `~/agent-shared/life/`.

## Protocol Objectives
- **Flat Token Cost**: Keep per-query token usage under ~3,500 tokens.
- **Prompt Caching**: Maximize the use of static index files for Anthropic/Google prompt caching.
- **Context Efficiency**: Load only what is necessary to answer the user's query.

## Navigation Steps

### 1. Identify Domain (Level 1)
Load the root index file: `~/agent-shared/life/_index.md`.
Analyze the user's query to determine which of the ten domains is most relevant.

### 2. Identify Leaf Files (Level 2)
Load the corresponding domain index file: `~/agent-shared/life/<domain>/_index.md`.
Identify the 1-2 specific leaf files that likely contain the answer.

### 3. Load Data (Level 3)
Load the identified leaf files (e.g., `~/agent-shared/life/finance/investments.md`).
Synthesize the answer for the user.

## Example Query
**User**: "When was the last oil change for the CR-V?"

1.  **Level 1**: Load `life/_index.md`. Identify **Vehicles** domain.
2.  **Level 2**: Load `life/vehicles/_index.md`. Identify `crv_2018.md` as the relevant file.
3.  **Level 3**: Load `life/vehicles/crv_2018.md`.
4.  **Answer**: "The last oil change was on 2026-04-15 at 45,200 miles."

## Rules for Agents
- **DO NOT** import the entire `life/` directory.
- **ALWAYS** follow the tiered loading sequence.
- **NEVER** write to the `life/` directory; it is human-curated via git.
