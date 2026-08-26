# Drop your source material here

Everything that describes what you want built goes in this folder. `/df-intake` reads **all of it** before asking you a single question.

Anything is fair game:

- the spec document itself — Markdown, text, Word, PDF
- screenshots, mockups, wireframes, sketches on a napkin (PNG, JPG)
- brand material: logos, colors, fonts, an existing style guide
- competitor screenshots or "make it work like this" references
- data samples: CSV exports, example API payloads, a schema dump
- emails, chat threads, meeting notes with decisions in them
- anything you'd otherwise paste into the chat

No naming convention required, no index to maintain. Subfolders are fine. If a file is
outdated or superseded, either delete it or put it in a subfolder named `old/` — Claude
reads that as "context, not requirements".

Two things that help a lot:

1. **Name files so their role is obvious** — `spec-v3.docx` beats `document(2).docx`,
   `mockup-dashboard.png` beats `Screenshot 2026-08-26 at 14.32.11.png`.
2. **If two documents disagree, say which one wins** — a one-line `NOTES.md` in this
   folder is enough. Otherwise Claude decides, and it will pick the more recent one.

The refined, actionable spec Claude produces from all this lands in `project/SPEC.md`.
This folder stays as you left it — it is the source material, and it is never edited.
