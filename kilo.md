---
type: Note
_organized: true
---

# kilo.md — Note placement & recategorization guide

This file is the Kilo-facing companion to `AGENTS.md`. It exists so that Hermes
and other Kilo sessions know exactly where to put new notes in this Tolaria
vault and how to recategorize ones that landed in the wrong place. General
Tolaria/frontmatter conventions live in `AGENTS.md`; this file focuses on
**folder placement**.

## Folder structure

> **Hard rule: every note goes in a subdirectory of the vault root.**
> Never create a loose note at the vault root. Pick the matching top-level folder
> for what the note is *about* (`projects/`, `customers/`, `todo/`, or `general/`),
> then the correct subfolder inside it. Customer notes go in `customers/`, project
> notes go in `projects/`, tasks go in `todo/`, everything else in `general/`.

The vault has four top-level folders. Place every new note in the folder that
matches what the note is *about*, not where you happen to be working:

| Folder | Contains | Example |
|---|---|---|
| `projects/<repo-name>/` | Notes about a single code repository / app / library — session handoffs, status notes, design docs. One subfolder per project, named after the repo in kebab-case. | `projects/billing_api/billing_api.md` |
| `customers/<customer-name>/` | Notes about a specific customer, tenant, or client engagement — requirements, meeting notes, deployment specifics. One subfolder per customer, kebab-case. | `customers/maryland/maryland-jul-13.md` |
| `general/` | Notes not tied to a specific project or customer — Tolaria type documents, generic how-tos, vault-wide references, templates. | `general/note.md`, `general/type.md` |
| `todo/` | Action items, task lists, reminders, follow-ups, checklists. | `todo/july-followups.md` |

## Decision rule (apply in order)

1. Is it about one code repo / app / library? → `projects/<repo-name>/`
2. Is it about a customer / tenant / client? → `customers/<customer-name>/`
3. Is it a task, TODO, checklist, or action item? → `todo/`
4. Otherwise (type doc, generic reference, vault-wide note) → `general/`

## When to create a note here

- **Session handoff / project status** for a repo → `projects/<repo-name>/<repo-name>.md`
  (one canonical status file per project; append a dated `## Earlier session — <date>`
  section instead of creating a second file for the same project).
- **Customer meeting or engagement note** → `customers/<customer-name>/<customer-name>-<short-date>.md`.
- **A task, TODO, or follow-up** → `todo/<short-description>.md`.
- **A Tolaria type document or vault-wide reference** → `general/<name>.md`.

## Placement rules

- One subfolder per project / customer, kebab-case, named after the repo or
  customer. Create the subfolder if it doesn't exist yet.
- Filenames are kebab-case, one note per file, first H1 = note title
  (Tolaria uses the H1 in lists, wikilinks, and search).
- Type documents (`type: Type` frontmatter) live in `general/`.
- The vault root should contain only: `AGENTS.md`, `CLAUDE.md`, `kilo.md`, and
  config (`kilo/`, `.gitignore`, etc.). **Do not leave loose notes at the root.**
- A note about a project deployed for a specific customer (e.g. OMSInt for Cobb)
  goes under `projects/<repo>/` if the focus is the codebase, or under
  `customers/<customer>/` if the focus is the customer relationship/deployment.
  When unsure, prefer `projects/`.

## Recategorization (when something is in the wrong place)

A note is misplaced if it sits at the vault root (other than the allowed root
files above) or in a folder whose subject doesn't match the note's subject. To
recategorize:

1. Identify the correct destination folder using the decision rule above.
   Create the subfolder if needed.
2. Move the file with `git mv <old> <new>`, preserving the filename and all
   frontmatter.
3. Wikilinks (`[[filename]]`, `[[Note Title]]`) resolve globally across the
   vault, so moves do not usually break links. Only update a wikilink if it
   explicitly used a relative path (rare).
4. If a move would create a filename collision (two status notes for the same
   project), do not keep duplicates: merge into one canonical file and append
   the older content under a dated `## Earlier session — <date>` section near
   the bottom.
5. Update `related_to` / `belongs_to` relationships only if the subject of the
   note changed, not just because it moved folders.

## What Hermes / Kilo should do

- **Always create notes in the correct subdirectory, never at the vault root.**
  Apply the decision rule above: about a code repo → `projects/<repo>/`, about a
  customer → `customers/<customer>/`, a task/TODO → `todo/`, otherwise → `general/`.
  Create the subfolder if it does not exist yet.
- When you notice a misplaced note (loose at the root, or in a folder whose
  subject doesn't match), recategorize it following the steps above.
- Keep shared agent instructions in `AGENTS.md`; this file stays focused on
  placement/recategorization.

## What Hermes / Kilo should avoid

- Do not create loose notes at the vault root — every note goes in a subdirectory
  matching its subject (`projects/`, `customers/`, `todo/`, or `general/`).
- Do not infer note type or meaning from folders — folders are an organization
  aid, not the source of truth (the `type:` frontmatter is).
- Do not keep duplicate status files for the same project; merge them.
