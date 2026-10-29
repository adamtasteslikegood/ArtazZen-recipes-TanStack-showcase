# Project working agreements

Standing instructions from the project owner for agents working in this repo.

## Default to action
- If a request is ambiguous between implementation and theory/discussion, lean toward
  implementing: make the edits, build the prototype or MVP, and save files.
- Prefer doing the work over describing what could be done.

## Always commit and push
- Commit and push after every meaningful change. Do not leave work uncommitted.
- Reason: dev boxes crash, especially old ones running parallel agent work. Anything not
  pushed can be lost.
- Push with `git push origin HEAD:main` from the owned worktree branch (do not rename or
  switch the owned branch).

## Related context
- Decisions live in `docs/adr/`. Sample content and schemas live in `docs/`.
- Do not commit secrets or internal fields (`ai_metadata`, `user_id`) from real exports
  without checking whether the repo is public.
