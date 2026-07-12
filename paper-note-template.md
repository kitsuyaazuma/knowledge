---
type: Template
title: Paper Note Template
description: Reusable metadata and section structure for paper reading notes.
tags: [papers, research, template]
timestamp: 2026-07-12T00:00:00+09:00
---

# Usage

Create one concept document per paper under [`papers/`](papers/) using the
following template, and add it to [`papers/index.md`](papers/index.md). Keep
the top-level section order stable, add method-specific subsections where
useful, and remove optional sections that are empty in the completed note.

Use the canonical publication page for `resource`, rather than a local PDF.
Use `pdf` for the exact HTTPS PDF used to read the paper, pinning its version
when possible. Omit `pdf` only when no directly accessible PDF exists; agents
must not guess a missing download URL.
Choose tags for the paper's topics; do not use tags for its venue or reading
status. Preserve reading questions as headings so that they remain searchable.
Distinguish claims made by the paper from personal interpretation or critique.
Make Reading Question answers explanatory rather than merely definitional: lead
with the direct answer, then add a concrete or worked example when it helps a
reader understand the mechanism, calculation, or distinction.
Format mathematical expressions for GitHub Markdown: use `$...$` inline and
`$$...$$` on separate lines for display math.

# Template

```markdown
---
type: Paper Note
title: <Paper title>
description: <One-sentence description of the paper and why it matters>
resource: <Canonical paper URL>
pdf: <Direct HTTPS URL for the exact PDF version>
tags:
  - <topic>
timestamp: <ISO 8601 datetime>
status: <to-read | reading | reviewed>
---

# Overview

## One-sentence takeaway

<!-- State the paper's central result in one or two sentences. -->

## Research question

<!-- Explain what problem the paper addresses and why it matters. -->

## Proposed approach

<!-- Explain the main method or system. Add descriptive subsections as needed. -->

## Main findings

<!-- Record the most important findings, including useful quantitative results. -->

## Contributions

<!-- State what is genuinely new in this work. -->

## Limitations and critique

<!-- Separate limitations stated by the authors from your own critique. -->

# Key Concepts

<!-- Explain concepts needed to understand the paper, one subsection per concept. -->

# Reading Questions

## <Question>

### Answer

<!--
Answer from the paper first, then add a concrete or worked example when useful.
Prefer examples with realistic inputs, intermediate steps, and outputs over
abstract restatements. Clearly label interpretation or inference.
-->

# Open Questions

<!-- Record unresolved questions or points worth revisiting. Omit if empty. -->

# References

1. [Paper](<canonical URL>)
2. [Code or project page](<URL>)
```
