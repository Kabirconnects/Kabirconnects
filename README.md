<div align="center">

<img src="https://readme-typing-svg.demolab.com?font=JetBrains+Mono&weight=600&size=26&pause=1200&color=7AA2F7&center=true&vCenter=true&width=660&lines=Hi%2C+I%27m+Himanshu;Full-stack+TypeScript+engineer;I+build+agent+runtimes+and+their+memory" alt="Typing SVG" />

<p><strong>Full-stack TypeScript engineer. I build agent runtimes and the memory layer underneath them.</strong></p>

<p>
  <a href="https://github.com/Kabirconnects"><img src="https://img.shields.io/badge/GitHub-Kabirconnects-181717?style=flat-square&amp;logo=github&amp;logoColor=white" alt="GitHub profile" /></a>
  <a href="https://github.com/Kabirconnects/NIMO"><img src="https://img.shields.io/badge/Project-NIMO-7AA2F7?style=flat-square" alt="NIMO repository" /></a>
</p>

</div>

---

## What I'm building

**[NIMO](https://github.com/Kabirconnects/NIMO)** is an experimental AI agent platform: an agent runtime, a sandboxed tool layer, an HTTP server, and web and terminal clients.

**PAME** (NIMO's Associative Memory Engine) is its memory subsystem, and the part I care about most. It is packaged as a standalone TypeScript library in `packages/memory`, so a host agent can use it without adopting NIMO's runtime, models, or interface.

Most agent memory is either a transcript appended to a context window, or text embedded into a vector store and retrieved by cosine similarity. Both discard structure: a fact looks the same as a question, a correction looks the same as a repetition, and a supported belief looks the same as a guess. PAME stores typed memory nodes with attached evidence, connects them with typed directional edges, and keeps writing, retrieving, and forgetting as separate governed operations.

> **Scope:** NIMO is the platform. PAME is its memory engine, not the whole product.

---

## Architecture

PAME is five layers, kept separate on purpose.

| Layer | Contents | Implementation |
| :-- | :-- | :-- |
| **Experience** | What actually happened: messages, tool calls, tool results, observations, documents | Append-only `experiences` table with source type and timestamp |
| **Memory** | What is worth retaining | 12 node types, each with confidence, importance, retention class, validity window, and an `evidence[]` array |
| **Association** | How memories relate | 17 typed relations, directional, weighted, with repairable graph degree |
| **Index** | How memories are found | SQLite FTS5, in-process BM25, entity matching, and 256-dimension hashed n-gram vectors |
| **Governance** | What may be stored, changed, or removed | Secret gate, owner scoping, explicit deletion, retention classes, audit log |

Everything lives in one SQLite database (`better-sqlite3`, WAL mode). There is no external embedding service, no vector database, and no graph database.

### How a recall works

1. **Candidate generation.** The query becomes tokens, entity names, and a 256-dimension hashed n-gram vector. Four independent ranked lists are produced: BM25, entity match, embedding cosine, and FTS5.
2. **Fusion.** Reciprocal Rank Fusion with `k = 60` merges the lists, because BM25 and cosine scores are not on comparable scales.
3. **Associative activation.** From the fused seeds, activation spreads along typed edges (`λ = 0.5`, per-hop decay `0.6`, at most 4 hops). Connected memories that share no query words can still surface.
4. **Deterministic ranking.** `0.35 × relevance + 0.40 × importance/degree + 0.15 × recency + 0.10 × confidence`. No hidden reranker and no non-deterministic tie-breaking.
5. **Assembly.** A bounded prompt block is built with evidence quotes. The model never receives the whole graph.

**Recall is read-only.** Retrieval writes no confidence, importance, degree, access counter, or edge, so repeating a question cannot make a memory more true. The hardening suite asserts this directly.

---

## Implementation status

I would rather publish an accurate status table than an impressive one. Verified against the repository on 15 November 2026.

| Capability | Status | Evidence |
| :-- | :-- | :-- |
| Hybrid retrieval | **Working** | Deterministic retriever with RRF, bounded graph activation, and truncation reporting |
| Rule-based ingestion | **Working** | Speech-act filtering (questions and instructions are not facts), assistant text is never treated as independent evidence, secrets blocked before capture |
| Contradiction handling | **Working** | Similarity is classified separately from conflict; contradictory memories are preserved and linked with a `contradicts` edge |
| Deletion | **Working** | One transaction per delete, idempotent: content, summary, embedding, evidence, edges, revisions, and FTS rows are cleared, with a non-sensitive audit record retained |
| Memory evolution | **Partial** | Conflicting and repeated evidence never inflates confidence; explicit correction lowers it. `evolve()` is implemented and tested but not yet wired into the ingestion path |
| Procedural skills | **Partial** | Candidate skills are distilled from tool-using turns; promotion requires a reported success. In my current database, 33 skills are still candidates and none has been promoted |
| Durable job queue | **Library and tests only** | Retry, backoff, dead-lettering, and stale-job recovery are implemented and tested, but no application code enqueues jobs yet |
| Dreamer, planner, bandit | **Library and tests only** | Offline consolidation, prerequisite planning, and UCB selection exist and are tested; they are not scheduled or used in the agent loop |

### Verification

Seven script suites, no test framework, isolated temporary databases:

```bash
pnpm --filter @workspace/memory test:all        # 312 of 314 checks pass
pnpm --filter @workspace/memory test:universal  # 62 questions across 6 axes
```

- The memory, ingestion, retrieval, conflict, deletion, and transaction suites pass. **Two checks currently fail:** the Chat and Plan permission assertions no longer match the shipped permission model, where memory writes are enabled in every mode and only deletion is gated. I am reporting the failure rather than deleting the test.
- The 62-question universal suite passes 100% across six axes. The corpus was written alongside the system, so it is a regression signal, not a blind benchmark.
- The 18-question retrieval benchmark scores Recall 100% and MRR 0.972 for the hybrid retriever, against 88.9% and 0.747 for vector-only. Same caveat: synthetic and self-authored.

---

## Design commitments

| Commitment | What it means in the code |
| :-- | :-- |
| **Evidence before belief** | Extraction records an experience id, evidence kind, quote, and observation time. Memories created directly through the agent's write tools carry no evidence and are not described as proven |
| **Contradictions are data** | Conflicting memories are preserved and linked, never silently overwritten |
| **Reading must not change belief** | Recall performs no writes; rehearsal is an explicit, separate operation |
| **Deletion is explicit** | Deletion clears derived storage in one transaction and keeps a non-sensitive audit record. This is logical deletion, not forensic erasure of SQLite pages or the write-ahead log |
| **Identity comes from the host** | The owner is established per request through `AsyncLocalStorage` and is never accepted from the model; multi-user mode denies a missing identity |
| **Deterministic over probabilistic** | Retrieval, ranking, conflict classification, and extraction are deterministic and testable, with no LLM in the memory hot path |

---

## Stack

![TypeScript](https://img.shields.io/badge/TypeScript-5-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-%E2%89%A520.9-339933?style=flat-square&logo=nodedotjs&logoColor=white)
![pnpm](https://img.shields.io/badge/pnpm-11.25-F46F2E?style=flat-square&logo=pnpm&logoColor=white)
![Turborepo](https://img.shields.io/badge/Turborepo-2.11-ED1C24?style=flat-square&logo=turborepo&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-better--sqlite3%2013-003B57?style=flat-square&logo=sqlite&logoColor=white)
![AI SDK](https://img.shields.io/badge/AI%20SDK-7-000000?style=flat-square)
![Zod](https://img.shields.io/badge/Zod-4-3E67B1?style=flat-square&logo=zod&logoColor=white)
![Express](https://img.shields.io/badge/Express-5-000000?style=flat-square&logo=express&logoColor=white)
![React](https://img.shields.io/badge/React-19-20232A?style=flat-square&logo=react&logoColor=61DAFB)
![Vite](https://img.shields.io/badge/Vite-8-646CFF?style=flat-square&logo=vite&logoColor=white)
![Tailwind CSS](https://img.shields.io/badge/Tailwind%20CSS-4-06B6D4?style=flat-square&logo=tailwindcss&logoColor=white)
![TanStack Query](https://img.shields.io/badge/TanStack%20Query-5-FF4154?style=flat-square&logo=reactquery&logoColor=white)
![Bun](https://img.shields.io/badge/Bun-%E2%89%A51.3-FBAC00?style=flat-square&logo=bun&logoColor=black)

**Toolbox:** TypeScript · Node.js · pnpm workspaces · Turborepo · SQLite and FTS5 · AI SDK · Zod · Express · React · Vite · Tailwind CSS · TanStack Query · Bun and OpenTUI · Git

---

## GitHub

<div align="center">

<img src="https://github-profile-summary-cards.vercel.app/api/cards/profile-details?username=Kabirconnects&theme=tokyonight" alt="GitHub profile summary" />

<img src="https://streak-stats.demolab.com?user=Kabirconnects&theme=tokyonight&hide_border=true" alt="Commit streak" />

<p>
  <a href="https://github.com/Kabirconnects?tab=followers"><img src="https://img.shields.io/github/followers/Kabirconnects?style=flat-square&amp;logo=github&amp;label=Followers" alt="Followers" /></a>
  <a href="https://github.com/Kabirconnects/NIMO"><img src="https://img.shields.io/github/stars/Kabirconnects/NIMO?style=flat-square&amp;label=Stars" alt="NIMO stars" /></a>
  <a href="https://github.com/Kabirconnects/NIMO/commits/main"><img src="https://img.shields.io/github/last-commit/Kabirconnects/NIMO?style=flat-square&amp;label=Last%20commit" alt="Last commit" /></a>
  <img src="https://img.shields.io/github/languages/top/Kabirconnects/NIMO?style=flat-square" alt="Top language" />
</p>

</div>

---

## Engineering principles

```ts
const principles = {
  claimOnlyWhatRuns: "If it is not wired into the application, it is a library, not a feature",
  testTheInvariant: "Read-only recall and single-transaction deletion are asserted, not assumed",
  structureOverScore: "Typed nodes and edges beat one more similarity threshold",
  provenanceFirst: "A memory you cannot explain should not outrank one you can",
  publishTheGaps: "Two failing checks are more useful than a green badge",
};
```

---

## Contact

The most useful place to reach me is the issue tracker on [NIMO](https://github.com/Kabirconnects/NIMO) or a discussion on [my profile](https://github.com/Kabirconnects).

NIMO is currently released under a proprietary, evaluation-only license. It is an experimental research project, and the repository README is the authoritative description of what works.
