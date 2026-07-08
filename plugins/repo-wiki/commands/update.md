---
description: Incrementally refresh the repository wiki from git changes since the last generation
---

Invoke the repo-wiki skill in **update** mode and follow its update workflow
exactly: locate the manifest, diff HEAD against its recorded commit, regenerate
only stale pages, then update the manifest and reference block.
