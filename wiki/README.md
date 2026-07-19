# Wiki drafts

Staging area for updates to the [GitHub wiki](https://github.com/vmanikes/go-project-structure/wiki).
These are `.md` only and are **not** part of the Go module — they don't affect `go build`.

The GitHub wiki is a separate git repo (`git clone https://github.com/vmanikes/go-project-structure.wiki.git`).
Paste each draft into the matching page there, then you can delete this folder.

| File | What to do with it |
|---|---|
| `Example-Orders-Domain.md` | **New page.** Create a wiki page "Example: The orders Domain, End to End" and paste in full. |
| `Code-Structure-additions.md` | **Append** the sections (config loading, versioning, testing) to the existing *Code Structure* page. |
| `Cross-Domain-DDD-addition.md` | **Append** the import-cycle example to the existing *Cross-Domain Entity Dependencies in DDD* page. |
| `Home-and-vocabulary-additions.md` | **Append** the "Start here" callout and hexagonal/clean table to the existing *Home* page. |

Existing pages are edited as *additions* rather than full rewrites because the drafts were written
from summaries of the live pages, not their verbatim source — appending avoids clobbering the
author's wording. Also worth a manual pass: several sub-pages threw transient "error while loading"
on fetch; confirm each renders after editing.
