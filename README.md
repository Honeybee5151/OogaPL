# tick-lang (working name)

A programming language only for game servers, and only for good server designs.
It makes the right design the only one you can write, so the compiler can optimise hard for it.

Status: **Stopped after Phase 1 (2026-11-08).** Miles decided not to continue. Main reason: AI assistants know Rust and C++ well but would know nothing about a new language, so it is easier to have AI write expert Rust directly. The research is kept for reference.

- `research/report.md` — prior art, performance evidence, the case against, and a recommendation.
- `research/notes/` — the raw evidence behind the report.

Phases:
1. Research (this phase) — then stop for Miles's go-ahead.
2. Design doc (`docs/design.md`), adversarially reviewed.
3. Prototype: parser, type checker, Rust code generator, tests, benchmark.
