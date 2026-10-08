---
name: writing-project-learnings
description: Use when studying an unfamiliar codebase and capturing what you learn as durable notes, when adding a project to a central learnings repository, when notes that cite code must still resolve months later, or when asked for an archify architecture, sequence, or workflow diagram to be saved under learnings.
---

# Writing project learnings

## Overview

Notes for every project live in one private repository, one directory per project.

**The repository stores notes and pins source. It never copies source.** A project's code is
referenced through a submodule gitlink, so a reader gets the exact revision the prose describes
without the notes repo carrying a line of someone else's code.

## Layout

The repository root **is** the index. Do not nest a `learnings/` directory inside it.

```
<repo root>/
  README.md            index: one row per project, with its pin
  <project>/
    README.md          entry point, reading order, the pin
    01-*.md … NN-*.md  notes in reading order
    VERIFICATION.md    what was actually run vs merely expected
    _src               submodule -> the project repo, pinned
  _shared/             only once something is genuinely shared
```

## Pin the source with a submodule

```bash
cd <learnings>
git submodule add <project-remote> <project>/_src
git -C <project>/_src checkout <full-40-char-sha>
git add .gitmodules <project>/_src        # then ask before committing
```

Already have the repo on disk? Avoid a second download:

```bash
git submodule add --reference /path/to/local/checkout --dissociate \
    <project-remote> <project>/_src       # --dissociate: no lasting link to the local checkout
```

Readers get the matching code with `git clone --recurse-submodules`, or
`git submodule update --init <project>/_src` for one project.

### Why a submodule and not a SHA written in the README

Agents reliably reject the submodule for a reason that is factually wrong. Rebut it:

| Objection | Reality |
|---|---|
| "It makes every clone drag down a huge tree" | **False.** `git clone` does not fetch submodules. Only `--recurse-submodules` does, and `--init <path>` fetches one project. A notes-only clone stays small. |
| "A dozen projects means a dozen submodules to manage" | They are independent. You init only what you are reading. |
| "A SHA in the README gets 90% for zero machinery" | It gets the revision but not the links. A README SHA cannot be clicked, and nothing enforces that the reader checked it out. |
| "It is another team's repo" | You store a 40-byte gitlink, not their code. Nothing is copied or republished. |

Real cost, state it honestly: submodule paths do not browse in the GitHub web view. Links resolve
in a local clone only.

The lighter alternative, if navigable links genuinely do not matter: record the pin in the
project README and reach the code with
`git worktree add --detach ../<project>-<shortsha> <sha>`, which leaves your main checkout alone. You then lose clickable links and keep prose citations.

## The link contract

**Every reference to code goes through `_src/`. Never `../`.**

```markdown
[kernel_scheduler_tlm.cpp](_src/src/compute/kernel_scheduler_tlm.cpp)
```

`../` looks like it should work when the notes are mounted inside the project checkout. It does
not. A directory symlink resolves **physically**, so from `<project>/learning/note.md` a `..`
lands in the learnings repo, not in the project. This breaks silently and at scale. `_src/`
resolves identically from a plain clone and through the symlink, because both reach the same
checkout.

Mount notes beside the code so you can write while reading:

```bash
ln -s <learnings>/<project> /path/to/<project>/learning
echo /learning >> "$(git -C /path/to/<project> rev-parse --git-path info/exclude)"
```

`.git/info/exclude` is local to that clone: nothing enters the project's tracked files, no pull
request needed. **Never add the learnings remote to a project checkout** — every branch there
carries that project's whole history, so one mistyped push publishes it.

## Note conventions

- Number notes in reading order. Studies outside the sequence take a descriptive name.
- **Write a note per question you had to answer, not per source file.**
- Head every note with the revision it describes, as a full 40-character SHA plus branch and
  date, in exactly this form (check 3 greps for it):

  ```
  Pin: <full-40-char-sha> (<branch>, YYYY-MM-DD)
  ```

  Short SHAs go ambiguous as a repo grows; the date and branch still locate a commit that was
  force-pushed away.
- Mark each non-obvious claim `VERIFIED` (you ran it) or `GUESS` (you read it and inferred). An
  unmarked guess is worse than no note.
- Quote 2–10 lines verbatim for anything load-bearing. This is mandatory, not decorative: it is
  the only part that still works when the pinned commit has been garbage-collected or rewritten,
  when the reader lacks access to the project remote, or when they are reading on the web.
- Say what the note does **not** establish — "code inspection only, no measured runs". Record
  unrun expectations in `VERIFICATION.md`, never mixed in with results.
- One copy only. If notes exist both in the project repo and the learnings repo, they will
  diverge and you will edit the stale one.

## Diagrams

Architecture, sequence, and workflow diagrams are notes too. Every rule above applies to them:
they go in `<project>/`, they carry the pin, and one copy only.

1. Read the code the diagram covers — entry points, core components, external dependencies,
   trust boundaries. Draw only what the code shows.
2. Invoke the `archify` skill and follow it. Do not use graphify or other knowledge-graph tools.
3. Keep it readable: 8–12 core components, one primary path. Extra detail goes in cards, not
   more edges.
4. Head the diagram's note (or its README row) with the `Pin:` line, like any note.
5. When the pin advances, revise the diagram in place. Git keeps the old version; a second copy
   would carry a stale pin and fail check 3.
6. To render or check the diagram, use the obscura headless browser if it is available. If it is
   not, or it fails, commit the HTML anyway and say so. Do not keep retrying.

## Before committing

Three checks. Run all of them.

```bash
cd <repo root>

# 1. nothing escapes through ../
grep -rn '](\.\./' <project>/*.md && echo 'LINK CONTRACT VIOLATION'

# 2. every relative link resolves (Markdown only; links inside diagram HTML are not checked)
grep -rhoE '\]\(([^)#]+)' <project>/*.md | sed 's/^](//; s/ .*//' \
  | grep -vE '^(https?:|mailto:)' | sort -u \
  | while read -r p; do [ -e "<project>/$p" ] || echo "BROKEN: $p"; done

# 3. every note's Pin line matches the staged pin
pin=$(git rev-parse :<project>/_src)
grep -L "^Pin: $pin" <project>/*.md
```

Check 3 must print nothing. Any file it lists has a missing or stale `Pin:` line: either the header
lied or the pin moved underneath the prose. Both mean the note describes code that is not what a
reader will get. It reads the staged gitlink, not `_src`'s working tree, so it checks the pin you
are about to commit.

**Ask before every commit and again before every push.** A yes to one is not a yes to the next.
Write the message with the `commit-messages` skill.

The learnings repo stands alone: no "re-rooted from" or other text about where commits came
from. Never commit secrets, keys, or share links that carry a key (`?sk=`).

## Common mistakes

| Mistake | Fix |
|---|---|
| Links written as `../src/...` | Route through `_src/`. `..` escapes the notes repo. |
| Copying project source in "so it works standalone" | Pin it. A copy silently rots against the real repo. |
| Short SHA in the note header | Full 40 characters, plus branch and date. |
| Notes committed inside the project repo | Symlink plus `.git/info/exclude`. The project repo stays untouched. |
| Advancing the pin without re-reading | Re-verify in the same commit, or leave the old pin. |
| Committing or pushing unasked | Ask each time. |

## Red flags

- About to `git push` a project branch that contains notes
- About to add the learnings remote to a project checkout
- Two copies of the same note in different repos
- A note that cites code but no pin records which revision
- "The links work on my machine"
