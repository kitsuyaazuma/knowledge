---
type: Agent Guide
---

# Agent Guide

Follow [Open Knowledge Format v0.1](SPEC.md) as the source of truth.

- **Orientation**: Read [index.md](index.md) first.
- **Scope**: Keep changes minimal, spec-faithful, and avoid custom directory conventions.
- **Updates**: Update the relevant concept file and the nearest [index.md](index.md).
- **Paper notes**: Add paper notes under [papers/](papers/). Before adding one, read and follow the [Paper Note Template](paper-note-template.md). Preserve its metadata fields and top-level section order; add or omit subsections when appropriate, and update [papers/index.md](papers/index.md).
- **Paper sources**: Paper PDFs are fetched on demand and must not be committed. When a question requires details beyond a paper note, run `scripts/fetch-paper <paper-note>` and inspect the returned PDF path. Do not download a paper when the note is sufficient, and do not invent a download URL when `pdf` is absent.
