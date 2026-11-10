# Hindsight interface between Carly and tern-obsidian

ready: true

Notification record: 2026-10-26 22:06, after the plugin hot-reloaded, the 7 Hindsight exports were verified as registered through Carly's `lua` tool (`type()` is function for all of them), and `hindsight_recall({limit=1},'/tmp/tern-hs-vault')` returned document_id `tern-hs-vault/alpha.md` state current. The intercom target session had disconnected, so this file serves as the delivery notice to Carly.

Hindsight stores a knowledge copy of the vault. The plugin does not hook into Carly's session memory and does not automatically retain user or assistant conversations. Each vault uses the configuration in its own `.obsidian/plugins/hindsight/data.json` (inherited read-only); Tern's overrides are written to the `hindsight` key of `<vault>/.obsidian/tern-obsidian/settings.json`, and sync state is written to `<vault>/.obsidian/tern-obsidian/hindsight/<binding>/`. Both travel with the vault and do not go into Tern's kv. The endpoint and bank must be present explicitly; the agent memory bank is not used.

Locks and heartbeats stay in Tern's plugin-data (machine-local) and do not take part in vault syncing: sync services have no strong consistency, a remote pid is meaningless, and putting locks in a synced directory would only shuttle temporary files back and forth. The index, pending operations, and job history still travel with the vault.

The index is compatible with the Obsidian plugin: the same `sha256:` raw-byte fingerprint, `vault/path` document IDs, `update_mode: replace`, and synthesized `vault:`/`folder:`/`created:`/`updated:` tags. A repeated sync overwrites the same document and does not create duplicate memories. Tern never deletes inherited documents. The risk is that when the two sides are configured differently (for example different observationScopes or scope), whichever side writes last determines the document content.

## Actual calls

| Actual call | Returns and behavior |
| --- | --- |
| `hindsight_status(root?, job_id?)` | Read-only configuration and sync status. Before the first sync, baselineInherited counts the reference syncIndex. Pass a job_id to track that same job. |
| `hindsight_sync(opts?, root?)` | opts is `{force?,job_id?}`. prepare returns `{job_id,root,bank,state,launch}` immediately, and the Tern side owns the background process. status does not advance the job. |
| `hindsight_recall(query, opts?, root?)` | Returns facts, citation IDs, expanded observation source_facts, sources, and next_offset. |
| `hindsight_reflect(question, opts?, root?)` | Returns text, based_on, sources, structured, and usage. |
| `mental_model(action, opts?, root?)` | Read: list/get/query/operation. Write: create/update/refresh/delete. |
| `knowledge(action, opts?, root?)` | Read: tree/search/get/operation. Write: folder/create/update/refresh/delete. |
| `hindsight_config(patch?, root?)` | Reads the configuration with the key hidden. patch explicitly modifies Tern's overrides; null clears the corresponding override. |

The configuration fields are apiUrl, apiKey, bankId, includeFolders, excludeFolders, prefixDocId, defaultBudget, syncOnEdit, observationScopes. The API key never appears in results or logs. recall accepts types, budget, max_tokens, tags, tags_match, tag_groups, scope, folder, prefer_observations, include_chunks, include_entities, include_source_facts, min_scores, offset, limit; pagination covers the server-side retrieval set of this request, and server-side reranking may change the ordering of subsequent requests. reflect accepts budget, max_tokens, tags, tags_match, tag_groups, scope, folder, fact_types, response_schema, exclude_mental_models, exclude_mental_model_ids, trace. mental_model query requires id and query; it hands the model's cached body to reflection and does not refresh implicitly. The model management options are id, name, source_query, query, tags, tags_match, trigger, max_tokens, limit, offset, detail, operation_id, budget, scope; the knowledge management options are id, name, source_query, query, tags, tags_match, tag_groups, trigger, max_tokens, parent_id, limit, scope, operation_id. When refresh/create returns an operation_id, read the result with the operation action.

The source structure is `{root,rel,path,document_id,citation_id,chunk_id,state,line?}`, where state is current, stale, excluded, or unresolved. An aggregated observation that has no document_id of its own stays unresolved, and sources lists the real original fact sources; citation IDs are kept verbatim.

## Sync semantics

Sync selects Markdown only; the scope is the intersection of the reference includeFolders/excludeFolders and Tern's allow/deny, with deny taking precedence. Document IDs keep the vault-name/path form. The reference syncIndex is used read-only to skip content already ingested; Tern only deletes documents it retained itself, and inherited documents always remain owned by Obsidian. force only resends the current scope and does not clear the bank.

Jobs are owned by the Tern process: prepare atomically takes the lock, writes the job record, and releases the lock; runSync runs under an exclusive `wx` lock with a 10-second heartbeat; after the process dies, the lock is judged stale by its ISO timestamp. Each retain's operation UUID is persisted to disk before the POST, and when the acknowledgement is lost the same UUID is resent, so no duplicate update is produced. `submitted` means accepted by the server, `ingested` is counted only when GET /operations/{id} returns completed, and `tracked` is the number of documents committed to the index. The polling budget is 60 seconds; on timeout the state is incomplete with the pending count noted, and it is never marked done. A failed job keeps its pending operations; resume the same job with `hindsight_sync({job_id},root)`. Jobs do not cross bindings: after the endpoint/bank/prefix changes, the old job_id cannot be reused.

When syncOnEdit is on, a file watcher is started (4-second debounce, cross-VM lock, configuration re-read every 10 seconds); starting the watcher does not upload the whole vault. When the configuration is missing or syncOnEdit is off, it returns `{state:'disabled'}` immediately.

## Acceptance evidence

Real vault, read-only (Alexandria, bank obsidian-alexandria): status baselineInherited=3487, referenceLastSyncAt correct; recall("causal emergence",{budget="low",max_tokens=1200,limit=2}) expands to papers/abstracts/emergence-causality-complex-systems-survey-abstract.md state=current; reflect returns an overview and 6 current sources including blogs/context/causal-emergence-community-context.md; mental_model list and knowledge tree pass. Carly read the original text back to check it, and the 22:00 re-verification of this round passed.

Isolated vault /tmp/tern-hs-vault (bank tern-obsidian-test):

| Operation | Evidence |
| --- | --- |
| Add alpha.md | job done, submitted=1/ingested=1, recall hits document_id tern-hs-vault/alpha.md state=current |
| Add gamma.md | job done, submitted=1/ingested=1, skipped=1 (alpha fingerprint unchanged) |
| Modify alpha.md | job done, submitted=1/ingested=1, resent with update_mode=replace |
| Delete gamma.md | job done, deleted=1, recall for ternhindsightgamma returns 0 hits |
| Fault: endpoint changed to 127.0.0.1:9 | job failed, submitted=0/ingested=0/failed=1, errors record ECONNREFUSED and pending 1, the failed state is kept as is |
| Retry after recovery | The same job_id resumes to done; ingested=1 after the server-side operation completes; the idempotent UUID prevents duplicates |
| Job interruption | hs-mvpbh9gq, submitted by Carly before 22:00, was resumed to done by the background process, submitted=1/ingested=1/tracked=1 |
| watcher | syncOnEdit=false returns {state:'disabled'}, producing no logs or uploads |

Isolated acceptance also created a smoke mental model, folder, and knowledge page and renamed them successfully via update; every operation receipt returned an operation_id.

## Known limitations

- The reference Obsidian plugin's body is not chunked; the whole body is a single retain item.
- A query against a mental model with a large cache makes for a heavy reflection; a 60-second timeout returns a failure rather than faking success.
- If no process claims the job within 60 seconds after prepare, the job is treated as interrupted and can be resumed.
- The real vault Alexandria has not yet been synced with a Tern job; its recall/reflect read the content already ingested by the Obsidian plugin directly.
