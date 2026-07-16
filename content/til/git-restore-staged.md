---
title: "TIL: git restore --staged is the friendly unstage"
date: 2025-07-08
---
Instead of remembering `git reset HEAD <file>`, you can just say what you mean:

```bash
git restore --staged file.txt
```

It unstages without touching your working-tree changes. Reads better too.
