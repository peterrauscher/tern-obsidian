# Tern Obsidian

This plugin lets Tern recognize Obsidian vaults (directories containing `.obsidian/`) and open notes, Bases, and JSON Canvas files in native reader blocks. Body parsing reuses remark, remark-gfm, and remark-wiki-link. YAML parsing reuses Bun's built-in `Bun.YAML.parse`.

## Installation

Requirements: Tern 0.9.8 or a compatible SDK version, and a Bun version that provides `Bun.YAML`. Local verification used Bun 1.4.2.

```sh
cd tern-obsidian
bun install --frozen-lockfile
tern plugin link "$PWD"
tern plugin list
```

The list should show `obsidian … ready`. Tern watches the plugin directory and reloads automatically when Luau files change.

## Usage

Open Tern inside a vault directory, or open a `.md`, `.base`, or `.canvas` file inside a vault from Files.

| Entry point | Behavior |
| --- | --- |
| Open file… | Fuzzy-searches file names, aliases, and tags in the current vault; selecting a result opens the note. |
| Obsidian: Open note | Opens the vault file picker. When the current block does not belong to a vault, you can enter an absolute vault path. |
| Obsidian: Open base | Picks and displays a `.base` file. |
| Obsidian: Refresh vault index | Rebuilds the index of the current vault. |
| Obsidian: Save Tern canvas as note | Picks a Tern UI canvas and saves its text, Markdown, tables, and code to `Tern Notes/` in the vault. A numeric suffix is added automatically when the file name already exists. |
| Obsidian: Save whiteboard as note | Saves the text and connections of the current whiteboard as a Markdown note. |
| Obsidian: Import canvas as whiteboard | Imports JSON Canvas nodes and connections into a Tern whiteboard. |
| Obsidian: Edit vault filters | Edits the per-vault allow/deny index filters. The panel floats at the bottom right of the current pane and is reused when already open. |
| Obsidian: Hindsight settings | Configures the per-vault binding and sync scope of the Hindsight twin knowledge base, and shows or starts sync jobs. |

At the bottom of a reader block, **Edit source** opens the original file and **Refresh** re-reads the note. The `e` and `r` keys map to these two actions. A refresh after editing the source reads the new content. Opening through the `tern open` CLI keeps Tern's native file editing behavior, which suits `EDITOR='tern open --wait'`.

The note picker sorts by the character count of the longest contiguous matched run, descending; ties are sorted by vault-relative path. Scattered-letter matches still take part in the search. The picker temporarily takes over the current block's position and restores the original block after cancelling or selecting.

## Vault filters

`Obsidian: Edit vault filters` floats the per-vault filter panel at the bottom right of the current pane; when already open, the same panel is reused instead of stacking tabs. The panel is a Tern `prefs` settings page, capped at 640px wide with 14px/16px padding. Click the `Pattern` input row and press Enter to add to the current target list; `Add to` switches the target with an Allow/Deny segmented control, and `Remove` on the right of each list entry deletes it. The page header shows the number of currently visible files.

| List | Semantics |
| --- | --- |
| Allow | When non-empty, only matching files are listed; when empty, all files are listed |
| Deny | Matching files are not listed; Deny wins when a file matches both lists |

Patterns match vault-relative paths. `**` crosses directories; `*` and `?` match within a single path segment. A pattern without `/` matches both file names and directory prefixes, so `raw` covers `raw/…` and `*.canvas` covers canvases at any depth.

Filters apply only to the index: the file picker, Bases tables, tags, outgoing links and backlinks, and Carly's `find`, `links`, and `vault`. Filtered-out files can still be opened with `read` and the file editor, consistent with Obsidian's own ignore filters. Changes take effect on the next open or index Refresh.

The configuration is stored inside the vault under the `filters` key of `<vault>/.obsidian/tern-obsidian/settings.json`. It travels with the vault and does not go into Tern's kv.

## Carly read interface

The window half registers five read-only exports that Carly calls directly from the `lua` tool, with no need to spawn a subprocess or run its own index. The root defaults to the vault that the focused pane belongs to; every export also accepts an explicit `root`, with paths coming from `vaults()`.

| Call | Returns |
| --- | --- |
| `await(plugins.obsidian.vaults())` | The vault root of the focused pane (nil if none), plus every vault path recorded by Obsidian |
| `await(plugins.obsidian.vault(root?))` | Vault root, note count, and the first 20 index warnings |
| `await(plugins.obsidian.find(query, limit?, root?))` | Case-insensitive substring match over paths, file names, aliases, and tags; `total` is the full hit count, `matches` holds at most 50 entries, `limit` defaults to 20 |
| `await(plugins.obsidian.read(rel, from?, lines?, root?))` | Reads the body paged by line; `rel` may be a vault-relative path or an absolute path inside the vault; defaults to 200 lines, at most 12 KB per page, `next` is the starting line of the next page |
| `await(plugins.obsidian.links(rel, root?))` | Outgoing links and backlinks, at most 80 each; files removed by ignore filters have `excluded` set to true and their own outgoing links are not resolved |

`tern.carly.context` also registers a hint so that Carly can see which vault the current pane belongs to. The five exports are read-only: file changes are made with Carly's own file tools or handed to an agent dedicated to managing the vault.

## Hindsight twin knowledge base

`Obsidian: Hindsight settings` opens a settings page styled like the filter panel: binding (API URL, bank, key), sync scope (include/exclude, document ID prefix, auto-sync on edit, observation scopes), and sync status (tracked/inherited document counts, latest job, manual Sync).

All configuration and state is bound to the vault and never written to Tern's kv:

| Path | Contents |
| --- | --- |
| `<vault>/.obsidian/plugins/hindsight/data.json` | The Obsidian plugin's own settings and `syncIndex`; inherited read-only, Tern never writes back |
| `<vault>/.obsidian/tern-obsidian/settings.json` | Tern's overrides (`hindsight`) and vault filters (`filters`) |
| `<vault>/.obsidian/tern-obsidian/hindsight/<binding>/` | The binding's index, pending operations, and job history; `binding` is a short hash of `root\|endpoint\|bank\|prefix` |
| `<state>/plugin-data/obsidian/hindsight/<binding>/` | Locks and heartbeats only (`lock.json`, `watch.lock.json`); machine-local, not synced with the vault |

Sync selects Markdown only, taking the intersection of the reference include/exclude and Tern's allow/deny. The reference plugin's syncIndex is used read-only to skip content already ingested; Tern only deletes documents it retained itself. Jobs are owned by the Tern process; prepare returns a job_id immediately, `submitted` means accepted by the server, and `ingested` means the operation completed. Failures, interruptions, and retries are all kept in the job record.

Seven exports are added on the Carly side: `hindsight_status`, `hindsight_sync`, `hindsight_recall`, `hindsight_reflect`, `mental_model`, `knowledge`, `hindsight_config`. See `carly-hindsight-contract.md` for signatures and semantics. Every fact from recall and reflect carries a citation ID and a source (current/stale/excluded/unresolved), and aggregated observations are expanded to their original facts.

## Format support

- Links: `[[Note]]`, `[[folder/Note]]`, `[[Note|Display name]]`, aliases, relative Markdown links, heading references, and `#^block` references.
- Embeds: Markdown notes, heading ranges, paragraph blocks, and images. Images are rendered from real bytes sent by the host through `cx:blob` and can be clicked to enlarge. Circular note embeds show a notice. Attachments get a file link.
- Properties: reads YAML frontmatter and displays text, numbers, booleans, lists, and objects. Wikilinks inside properties take part in the backlink index.
- Tags: frontmatter `tags` and inline `#tag`, including Chinese and nested tags. Code, escaped text, wikilink internals, and purely numeric tags are excluded from the tag index. Clicking a tag shows the related notes.
- Index: shows the table of contents, outgoing links, and backlinks. The asynchronous scan runs in a separate Bun process, and the cache detects updates by file size, modification time, and status change time. Deleted files leave the index on the next scan.
- Bases: table views, global and per-view filters, `and/or/not`, comparison and arithmetic expressions, common file/string/list functions, formulas, column order, column names, sorting, grouping, and row limits. Each table shows at most 200 rows along with the total count. Inline `base` code blocks are rendered too.
- Canvas: JSON Canvas text, file, link, and group nodes and their connections are shown as native cards. Whiteboard import keeps positions and connections; file nodes show the file name.

## Limitations

The reader block provides a preview; the original file is edited in Tern's file editor. Heading and block references jump to the corresponding line of the original file. A file's name and its duplicate aliases each form only one candidate. When different files share a name and the target cannot be determined uniquely, the reader block shows an ambiguity notice.

Bases currently provides read-only views. Summaries, the full set of Obsidian formula functions, third-party views, and Dataview queries are not implemented. Unsupported expressions show an error in the corresponding view. The JSON Canvas card reader offers no spatial editing, and whiteboard import does not keep every visual style. Whiteboard-to-Markdown keeps text and connections but not shape geometry.

Window commands and the file picker enhancements are meant for local vaults. Opening remote files falls back to Tern's native handling. Files not yet downloaded from cloud drives such as OneDrive may fail to read, and the reader block shows the specific error. Invalid YAML properties are never rewritten automatically.
When a note is opened through a symlinked path, the plugin resolves the file to its real path before determining its vault-relative location.

## Verification

The implementation passes `luau-lsp analyze` from the local SDK. Verification in a real Tern window covered: a real vault with several thousand notes, a `.base` table view, Open file search and selection, wikilink click navigation, heading/block reference resolution, property and tag display, reading the Markdown content after a Tern canvas export, and the two `prefs` panels for filters and Hindsight.

Short-link verification covered unique file names, duplicate aliases, same names in different directories, same-directory precedence, paths, and anchors. The BGAI close-reading report in Alexandria renders through `renderNote` with no ambiguity notice. The bottom buttons are fully visible in reader areas of 396 × 573, 396 × 1240, and 813 × 595 px, and both actions were verified by clicking. The fixed match-sorting samples have contiguous lengths of 9, 9, 6, and 1, and ties stay consistent across 10 repeated runs; in the real picker, `attention` returns 89 results.
Path verification covered a real OneDrive symlink alias path, the physical CloudStorage path, opening relative paths from different working directories, and rejection of paths outside the vault.

Reports containing images keep their `marks` link metadata when split into Markdown fragments. In the real UI, the "BGAI abstract" link in the BGAI close-reading report produces an `.sf-link` with a target address; Command-clicking it opens `papers/abstracts/broadcast-gated-attention-abstract.md`, showing the corresponding paper title and official abstract.

Hindsight side: the real vault (Alexandria, `obsidian-alexandria`) passed read-only acceptance, with `baselineInherited` counting 3487 inherited documents and recall and reflect citations readable back to the original text; the isolated vault (`tern-obsidian-test`) passed add, modify, delete, the ECONNREFUSED fault staying failed, idempotent retry with the same job_id, background resume after interruption, and the watcher returning disabled when `syncOnEdit=false`.

## Changelog

### Unreleased

- Fixed duplicate candidates for short wikilinks while keeping the ambiguity notice when different files share a name.
- Fixed notes opened through a symlinked path being misjudged as outside the vault.
- Fixed image splitting dropping Markdown link metadata, restoring body wikilink click navigation in notes with images.
- Fixed the horizontal layout and clipping of the action area at the bottom of the reader block; buttons keep the same height and bottom space is reserved.
- The note picker uses Tern's controlled picker, sorted by longest contiguous matched run descending, with ties sorted by path.
- Carly read interface: `vaults`, `vault`, `find`, `read`, `links`, all read-only; the root can come from the vault of the focused pane.
- Vault filters: the `obsidian.filters` block edits allow/deny globs on a `prefs` settings page, with configuration stored in the vault at `.obsidian/tern-obsidian/settings.json`.
- Hindsight twin knowledge base: the `obsidian.hindsight` settings page, the background sync job engine, and the seven Carly exports `hindsight_status`, `hindsight_sync`, `hindsight_recall`, `hindsight_reflect`, `mental_model`, `knowledge`, `hindsight_config`.
- Vault root resolution fallback: when the focused pane is not inside a vault, the most recently used vault is used instead, then the picker over Obsidian's vault list.
- Fixed: an empty cwd is no longer treated as a vault path to search under; the floating panel reuses the already open pane instead of stacking tabs.

## References

- [Tern Plugin SDK](https://docs.stencil.so/tern/)
- [Obsidian official help source](https://github.com/obsidianmd/obsidian-help)
- [Bases syntax](https://obsidian.md/help/bases/syntax)
- [JSON Canvas 1.0](https://jsoncanvas.org/spec/1.0/)
- [remark-wiki-link](https://github.com/landakram/remark-wiki-link)
