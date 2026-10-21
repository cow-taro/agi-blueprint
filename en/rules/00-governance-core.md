# Governance core

This file is the authoritative top-level rule. It applies before every output and judgment.

## 0. Compass (top objective)

<!-- ▼▼▼ Enter SETUP-CHECKLIST.md Section A item A1 (Top objective (compass)) here ▼▼▼ -->

(Enter, in 1–5 lines, the purpose for which this project exists)

<!-- ▲▲▲ End of entry ▲▲▲ -->

- Whenever a judgment gets difficult, always return to this section. The direction of implementation, investigation, and fixes is decided from "what is needed to fulfill this purpose".
- Do not reason from the symptom in front of you alone. First picture where the feature sits in the users' business, then narrow down the cause and the scope of the response.
- Never paraphrase, summarize, or move this section. Keep it pinned at the top.

## 1. Philosophy: from knowledge volume to reasoning quality

- Do not dump raw code or unrelated material into a huge context (wasteful full scans). Full scans scatter attention and induce hallucination. The authoritative source is "uncorrupted, concentrated md", governed through unique references and evidence-based inference.
- Long thinking (Thinking) is the time of self-applying the rules (a logical audit). Never skip verification processes because of slowness.

## 2. Allowed and forbidden forms of references

### Forbidden (absolute)

- References by line number or physical address (`A-1`, `§3.2`, number ranges, etc.)
- Symbol anchors, bare numbers, or code words that cannot be resolved without guessing the file or context
- Undefined abbreviations
- Using common nouns as if they were proper nouns (e.g., writing a nickname someone coined, like "the ledger", "the system table", "category D", as if it were an official name. A name that becomes unique only after the reader guesses the file or context is not a proper noun)
- Slipping around a banned abbreviation (replacing a banned form with another form that is equally or more ambiguous)

### Allowed

- Real names unique across the repository: file paths, method names, skill names, constant names
- For sections, the full heading text (with the file path when needed)
- Example: `rules/01-workflow-gates.md`, the "3. Approval before edits" section

### Decision rule

Never write a reference that you (the AI) cannot resolve without guessing files or context. The point of banning abbreviations is not to reduce characters — it is to **prevent mix-ups**.

## 3. Design of the unique-reference graph

- Link documents to each other, like a mesh, using permanently unique primary keys (file names, full specification headings, method names).
- If an existing md already describes it, that description is authoritative. When writing something new, verify the reference target exists before writing it.
- Never write a sentence whose reference target you cannot pin down. Do not leave unresolvable abbreviations behind fake labels like "unresolved".

## 4. No raw code dumps; exact-match direct loading

- Never pour the full text of raw code into the context. "Read broadly and get a feel for it" is forbidden.
- Open code only at these times:
  1. When needed (once the investigation target has been identified from md)
  2. Immediately before implementation (load the file to be changed by exact path match before editing)
- When searching, use search (symbol search, full-text search) to identify the "location", then read only the necessary parts by exact match.

## 5. Topology stability of documents

- As a **result** of the strict reference rules, the string layout (topology) of documents is fixed, and a high cache hit rate follows. This is a designed outcome, not an accident.
- Do not casually reorder sections, rephrase headings, or normalize wording. It breaks references and lowers cache efficiency.
- Limit document updates to "when the result or content changes". Do not make small edits that change nothing (e.g., absorbing spelling variations).
- When moving or refactoring md files, never skimp on the time to maintain unique references. Search every md referencing the old location and sync the related md.

## 6. Escalation of repeated failures

- When the same failure happens three times, escalate it: project-specific traps go into the premises file (duplicated from `templates/project-premises-template.md`); reusable general procedures become a new skill. Do not create new numbered rule files on an ad-hoc basis
- Propose premises-file and skill additions to the user before making them. Do not skill-ize one-off or temporary judgments. Keep the always-loaded rules minimal; occasional content goes into skills.
