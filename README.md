# Zooko's software engineering style

A book of other people's descriptions of Zooko's software engineering
style: how he thinks about simplicity, communication, contracts,
measurement, and performance, as the people who work with him have
written it down. Zooko is its subject, not its author; each chapter says
who described the style and where the description comes from.

## Chapters

1. [Philosophy, Strategy, Style: Ode's description](chapters/01-ode-philosophy-strategy-style.md),
   as the shared part of the BLAKE3 servil, bench-hashes and Ode `AGENTS.md`.
2. [Style Guides: the bench-hashes version](chapters/02-bench-hashes-style-guides.md),
   an earlier form of the same description.
3. [Working philosophy: Devon Jonte's notes](chapters/03-devon-working-philosophy.md),
   as one collaborator adopted and amended it.
4. [Existence versus use: mappings, page tables, and smalloc](chapters/04-existence-versus-use.md),
   a contrarian hypothesis, tested.
5. [Build the special-purpose tool: a thread pool, from first principles](chapters/05-special-purpose-tools.md),
   Zooko's lesson about general-purpose abstractions and multithreading, with a measured case.
6. [Case study: keep the memory, skip the freeing](chapters/06-case-study-keeping-memory.md),
   Zooko's existence-versus-use hypothesis for a newcomer: the costs, the experiments, b3sum, and answers to the usual objections.

## Adding a chapter

A chapter is one person's description, with its provenance at the top: who
described the style, where the text comes from (a repository and commit,
a message, a talk), and whether it is verbatim. Descriptions copied from
elsewhere stay verbatim; later versions are new chapters, so the book
keeps how the descriptions changed.
