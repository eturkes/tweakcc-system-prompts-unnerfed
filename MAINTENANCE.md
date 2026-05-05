# Maintenance

How to keep these un-nerfed prompts current when Anthropic ships a new Claude Code release.

## Quick version

Every Claude Code update can change the wording of any system prompt. When that happens, you re-extract fresh stock prompts, copy them into this repo, and run `apply-unnerfs.py` to replay the un-nerfs against the new text. The script handles most of it automatically. You only step in when upstream changed the exact wording that a rule targets.

---

## The apply-unnerfs.py script

Located at [`scripts/apply-unnerfs.py`](./scripts/apply-unnerfs.py). Stdlib-only Python, no pip install. Safe to run repeatedly.

The script reads every `.md` file in `system-prompts/`, matches each against a table of `(stock_text, unnerf_text)` rules, and does string replacement. For each rule, one of four things happens:

| Result | Meaning |
|---|---|
| **APPLIED** | Found the stock text, replaced it with the un-nerfed version. File rewritten. |
| **SKIPPED** | Un-nerfed text already present. Nothing to do. |
| **NORMALIZED** | No content change needed, but CRLF line endings were fixed to LF. |
| **FAIL** | Neither stock nor un-nerfed text found. Upstream drifted. Needs your attention. |

It also normalizes any accidental CRLF line endings back to LF.

### Flags

| Flag | What it does |
|---|---|
| *(none)* | Apply all rules to `./system-prompts/`. Writes to disk. |
| `--dry-run` | Report what would change without writing anything. |
| `--check` | Like `--dry-run`, but exits 1 if any rule would apply or fail. Good for CI. |
| `--only FILE` | Restrict to one filename (just the name, no path prefix). |
| `--verbose` | Show context on SKIPPED entries too. |
| `--dir PATH` | Target a different prompts directory. |

---

## Full workflow after a Claude Code bump

tweakcc won't overwrite files you've edited. When it sees a conflict between upstream changes and your local edits, it writes an HTML diff alongside the `.md` and leaves your version alone. That's reasonable behavior for a tool with no external source of truth. But since *this repo* is the source of truth for un-nerfed content, the cleanest approach is wiping the tweakcc directory first, getting a clean stock extraction, and replaying everything here.

> Use `npx tweakcc-fixed@latest` instead of `tweakcc` while upstream catches up. See [BACKGROUND.md](BACKGROUND.md#which-fork-to-use) for why.

1. **Update Claude Code** the normal way (installer, npm upgrade, whatever).

2. **Wipe `~/.tweakcc/system-prompts/`.**
   ```bash
   rm -rf ~/.tweakcc/system-prompts          # Unix
   # Remove-Item -Recurse -Force "$HOME\.tweakcc\system-prompts"  # Windows
   ```
   Leave the rest of `~/.tweakcc/` alone. `config.json`, hash files, and `native-binary.backup` need to survive.

3. **Re-extract with tweakcc.** `npx tweakcc-fixed@latest`. It sees the missing directory, reads the new binary, and writes fresh stock `.md` files with no conflicts.

4. **Copy the fresh stock into this repo.** From the repo root:
   ```bash
   cp ~/.tweakcc/system-prompts/*.md system-prompts/
   ```
   Don't commit yet. Run `git diff` first to see exactly what Anthropic changed upstream.

5. **Run the re-apply script.** `python scripts/apply-unnerfs.py`. Read the report.

6. **Fix any FAILs.** Open each failed file, find the passage the rule targets (the report quotes the first 200 chars as a search term), and compare the new upstream wording against the old. Usually the drift is cosmetic and you just need to update `stock` in the rule to match the new wording byte-exactly. If upstream removed the passage entirely, delete the rule and note it in the commit.

7. **Re-run until clean.** Keep running the script until the report shows only APPLIED, SKIPPED, and NORMALIZED. No FAILs.

8. **Check for new prompts.** `git status` will show untracked `.md` files for anything Anthropic added. Read each one and decide if it introduces a brevity nerf worth flipping. Some new prompts (structured JSON generators with word caps, for example) should stay stock. Use the [un-nerf thesis](README.md#the-un-nerf-thesis) to decide.

9. **Commit.** The updated `.md` files and any `apply-unnerfs.py` rule changes together, while the diff is still small and the context is fresh.

10. **Copy back to tweakcc.** From the repo root:
    ```bash
    cp system-prompts/*.md ~/.tweakcc/system-prompts/
    ```

11. **Patch the binary.** `npx tweakcc-fixed@latest --apply`.

12. **Restart Claude Code** and verify a representative un-nerfed prompt actually made it in by triggering the relevant behavior.

---

## Adding a new un-nerf

1. Read the `.md` file you want to change. Find the passage to flip.

2. Open [`scripts/apply-unnerfs.py`](./scripts/apply-unnerfs.py) and add a `Rule(stock=..., unnerf=..., description=...)` entry under the matching filename key in `RULES`. Create the key if the file is new.
   - `stock` must match what's in the file byte-exactly, including trailing whitespace and any template literals like `${VAR}`.
   - `unnerf` is the replacement. Write it in the same thorough-over-brief voice as the rest.
   - `description` is a short scannable label (e.g. `"tone body: flip 'short and concise' to 'thorough'"`).

3. Preview: `python scripts/apply-unnerfs.py --dry-run --only <filename>`

4. Apply: run without `--dry-run` once you're happy with the preview.

5. Verify: `git diff` should show only the replacement you expected.

6. Confirm idempotent: `python scripts/apply-unnerfs.py --check`

7. Commit the script change and the un-nerfed `.md` together.

---

## Understanding FAIL reports

A FAIL in the report looks like this:

```
system-prompts/some-file.md
  [FAIL    ] <rule description>
             Expected stock text (first 200 chars):
               'Briefly explain what ...'
             Expected un-nerf text (first 200 chars, for reference):
               'Explain thoroughly what ...'
             Neither was found in the file.
             Action: open the file and locate the passage.
```

The two quoted excerpts tell you both what you're looking for (the pre-drift stock) and what the result should be (the target un-nerf). The file path tells you exactly where to go.

Three common causes:

- **Upstream reworded the passage.** Anthropic tweaked the phrasing in a new release. Find the new wording in the file, update `stock` in the rule to match. The `unnerf` usually doesn't need to change unless the new wording is structurally different.
- **Upstream removed the passage.** The nerfed directive got deleted from the prompt. Delete the rule and note it in your commit.
- **Upstream replaced it with something neutral.** The brevity directive got swapped for a neutral or pro-thoroughness one. Delete the rule. The un-nerf isn't needed anymore.

---

## CI / pre-commit (optional)

`--check` exits 1 when anything would change, so you can wire it into a pre-commit hook:

```bash
python scripts/apply-unnerfs.py --check || {
  echo "Un-nerfs not fully applied. Run: python scripts/apply-unnerfs.py"
  exit 1
}
```
