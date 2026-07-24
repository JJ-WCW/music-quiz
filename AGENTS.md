# Codex instructions

Before inspecting, planning, or changing this project, read `CLAUDE.md` in full.
Treat it as the authoritative project brief, including its architecture,
invariants, testing requirements, deployment notes, and working conventions.

When generating, editing, validating, or uploading quiz content, also read
`QUIZ-RECIPE.md` in full before taking action. It is not required for ordinary
application code changes.

Do not modify `CLAUDE.md` or `QUIZ-RECIPE.md` unless the user explicitly asks
for changes to those files. If an application change makes either document
inaccurate, tell the user what documentation may need updating instead of
editing it implicitly.

Preserve unrelated working-tree changes. Before making edits, inspect
`git status`; before handing work back, run the checks required by `CLAUDE.md`
and report any remaining uncommitted files.
