# Technique Talks — instructor decks

Fifteen-minute Game Technique Talks for the engine-spec sections the instructor owns (S00–S03,
S10, S16, S18). One directory per section. Same source format as the lecture decks one level up,
and built by the same tool:

```bash
./.venv/bin/python build_pptx.py talks/s10-3d-backend/s10-3d-backend.md
```

This directory is deliberately below `slides/`: both `build_pptx.py`'s default glob and
`build_lectures.dart`'s `listSync()` are non-recursive, so nothing here enters the lecture JSON or
the Canvas export. The `.pptx` lands next to its markdown and is gitignored like every other deck.
