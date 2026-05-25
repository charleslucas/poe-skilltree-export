# Skilltree Patches Overlay

GGG publishes the official passive skill tree as JSON in this repository. It's the most authoritative public source, but the export lags real game state — sometimes by months. A node's stat line can change in a patch without the export being re-tagged.

This fork keeps the upstream `data.json` **untouched** to minimize merge conflicts when pulling new GGG releases. Corrections live in a companion file, `data_patches.json`, applied as an overlay at load time. Tools that consume the tree data merge the two files in memory; on-disk, the GGG export stays pristine.

## File: `data_patches.json`

Sits next to `data.json` in this repository. Optional — if missing, no patches are applied. Format:

```json
{
  "11730": {
    "stats_add": ["0.4% of Attack Damage Leeched as Life"],
    "verified_from": "in-game tooltip",
    "verified_date": "2026-05-25",
    "verified_by": "Memophage#4428",
    "note": "GGG 3.28.0 export missing this stat; PoB also missing it"
  }
}
```

### Patch operations

| Key | Effect |
|-----|--------|
| `stats_add` | Appends entries to the node's existing `stats` array. Use when GGG has SOME of the stats but is missing additions. |
| `stats_replace` | Replaces the node's `stats` array entirely. Use when stats have been re-worded or completely revised. |
| `name_replace` | Replaces the node's `name` string. Rare. |
| `flags_set` | Sets `isNotable` / `isKeystone` / `isJewelSocket` / `isMastery` to provided values. Rare. |

A single node entry can carry multiple operations; they're applied in the order listed above.

### Required metadata on every patch

- `verified_from`: one of `"in-game tooltip"`, `"PoB tree data"`, `"PoB lua_get_passive_detail"`, `"wiki"`, `"reddit/forum"`.
- `verified_date`: ISO date (`YYYY-MM-DD`) when the patch was verified.
- `verified_by`: PoE account handle of the player who confirmed (`AccountName#1234`).
- `note` (optional): why the patch was needed, or any caveats.

`verified_date` is **load-bearing** for the merge resolution policy (below). Don't omit it.

## When to add a patch

Add an entry when an in-game discrepancy with `data.json` affects analysis:

1. **Stat mismatch**: in-game tooltip shows stats the JSON doesn't have, or vice versa.
2. **New node**: a node ID exists in PoB's bundled data or in-game but not in this export.
3. **Renamed node**: the stats are right but the name in JSON is out of date.

## When NOT to add a patch

- **Visual / positioning differences** (sprites, x/y, orbit) — these don't affect analysis.
- **Differences attributable to in-game transformations.** Most importantly:
- **Timeless Jewel modifications.** Timeless Jewels (Lethal Pride, Glorious Vanity, Militant Faith, Brutal Restraint, Elegant Hubris) **add or replace** stats on small/notable nodes within their radius based on the jewel's seed. A discrepancy on a character with one of these jewels socketed is most likely the jewel doing its job, NOT a stale export.

  **The blank-line convention.** When a passive node tooltip shows stats both above and below a blank line, the lines below the blank line are jewel-added. The lines above are intrinsic to the node. Example: a `+10 to Strength` small node within Lethal Pride radius will show:
  ```
  # Strength
  +10 to Strength      ← intrinsic
                       ← blank-line separator
  +2 to Strength       ← Karui addition from the jewel
  ```
  This convention is identical between the in-game passive tree view and the PoB passive tree view. **Use PoB as the copy source** — the game tooltip is read-only (no text selection), while PoB's tooltip text can be copied. The verification protocol below assumes PoB.

  - **Checklist before patching:**
    1. Confirm the node is in the user's tree AND they report the discrepancy.
    2. List any Timeless Jewels socketed in the character.
    3. **Apply the blank-line test**: are the suspected "missing" stats above or below the blank line? If below, they're jewel-added — do not patch.
    4. **If still uncertain, the controlled-removal test is definitive**: in PoB, hover the node and copy the tooltip text. Then unequip the Timeless Jewel from its socket and copy the tooltip text again. Any lines that disappear in the second copy were jewel-added. Re-equip the jewel afterward to restore the original build state.
    5. Only proceed with the patch if the stat persists with the jewel unsocketed AND still differs from `data.json`.
- **Cluster Jewel internal passives** — these are dynamically generated; they're not in `data.json` at all, and they shouldn't be.
- **Tattoo overrides** — these belong in the build's `<Override>` XML element, not the tree data.

## Upstream merge policy

This fork tracks `grindinggear/skilltree-export` (added as `upstream` remote). When GGG publishes a new export:

1. `git fetch upstream && git merge upstream/main` — merge their new `data.json`.
2. **Conflicts on `data.json`:** accept upstream's version. GGG's new data is presumed authoritative.
3. **Audit existing patches:** for each entry in `data_patches.json`, compare its `verified_date` against the GGG release date (usually the merge commit date).
   - **Patch newer than GGG release:** likely still relevant — keep applying. Verify the underlying node now matches our patch (if so, retire). Otherwise leave the patch in place.
   - **Patch older than GGG release:** the upstream data may now be correct on its own. Re-verify against in-game tooltip before keeping the patch. If GGG now matches, **delete the patch entry**.

The rule is: **most recent verification wins.** GGG's release is an implicit verification at the release date; our patches carry explicit dates. Don't let stale patches override fresher upstream data.

### Conflicts inside `data_patches.json`

If two contributors patch the same node in different branches and we hit a merge conflict on `data_patches.json`, **accept the entry with the more recent `verified_date`**. Older entries are obsolete. Git's three-way merge won't enforce this — it's a manual resolution policy. Don't try to merge stats across two competing patches; pick the newer one wholesale.

## How patches are consumed

Any tool that reads the tree data should merge `data.json` + `data_patches.json` in memory at load time. The `poe_mcp_suite` repo provides an MCP tool (`get_tree_node`) that does this; standalone scripts should implement the merge inline (~10 lines of Python).

A simple Python reference implementation:

```python
import json, os
def load_tree(repo_dir):
    with open(os.path.join(repo_dir, 'data.json')) as f:
        data = json.load(f)
    patches_path = os.path.join(repo_dir, 'data_patches.json')
    if os.path.exists(patches_path):
        with open(patches_path) as f:
            for nid, p in json.load(f).items():
                node = data['nodes'].get(nid)
                if not node: continue  # patch references missing node; log warning
                if 'stats_add' in p:
                    node.setdefault('stats', []).extend(p['stats_add'])
                if 'stats_replace' in p:
                    node['stats'] = p['stats_replace']
                if 'name_replace' in p:
                    node['name'] = p['name_replace']
                if 'flags_set' in p:
                    node.update(p['flags_set'])
    return data
```

## Contributing patches

If you find a discrepancy:

1. Verify against in-game tooltip (most authoritative).
2. Check the "When NOT to add a patch" section above — confirm it's not a Timeless Jewel transformation.
3. Add an entry to `data_patches.json` with full metadata.
4. Open a PR. Pre-merge sanity check: `verified_date` is today's date, `verified_by` is set, and the operation is appropriate (`stats_add` for missing stats, `stats_replace` only when the entire stat block has changed).

Patches verified with high confidence and broad applicability are also good candidates to submit upstream to https://github.com/grindinggear/skilltree-export as PRs or issues. If GGG accepts them, retire the local patch on the next merge.

## Sibling fork

- Atlas tree data with the same patches overlay: https://github.com/charleslucas/poe-atlastree-export
- Original GGG upstream: https://github.com/grindinggear/skilltree-export
