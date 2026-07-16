---
title: "TIL: jq's recursive descent operator"
date: 2025-07-13
---
`jq` has a `..` operator that recursively walks every value in a document. Combined with `select`, it's a quick way to find a key no matter how deeply it's nested:

```bash
echo '{"a":{"b":{"id":42}}}' | jq '.. | .id? // empty'
# => 42
```

I'd been writing out full paths by hand for years.
