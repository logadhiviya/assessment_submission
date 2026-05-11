# Part 3: Prompt Preparation

**Selected PR:** [#3279 — Add parentwork plugin](https://github.com/beetbox/beets/pull/3279)  
**Repository:** [beetbox/beets](https://github.com/beetbox/beets)

---

## 3.1.1 Repository Context

Beets is a command-line music library management tool written entirely in Python. Its core job is to help users organise their personal music collections by automatically tagging audio files with correct metadata. It does this by comparing existing track information against the MusicBrainz database — a large, community-maintained catalogue of music releases, recordings, and artists — and then correcting or filling in any missing details like album name, track number, release year, and so on.

The repository is structured around a powerful plugin system. The core beets package handles things like the database, file operations, and the CLI framework, but almost every user-facing feature is implemented as a plugin located in the `beetsplug/` directory. This makes it easy for contributors to add new functionality without touching core code. Users enable only the plugins they want in their configuration file.

The intended users of beets are technically inclined music enthusiasts who want full control over how their music library is organised and tagged. It is not a graphical application — it is entirely driven from the terminal, and users who get the most out of it tend to be comfortable editing configuration files and running shell commands. Many users also write their own plugins in Python to automate tasks specific to their collection.

The problem domain is digital music collection management, with a particular focus on metadata accuracy and library organisation. Beets sits at the intersection of file management, API integration (MusicBrainz, Discogs, etc.), audio format handling, and database management — all through a clean Python interface.

---

## 3.1.2 Pull Request Description

This pull request adds a new optional plugin called `parentwork` to the beets plugin ecosystem. The plugin is specifically designed for people who have classical or jazz recordings in their music library, because those genres have a more complex organisational structure than pop music.

The problem it addresses is this: in MusicBrainz's data model, a recording (a specific performance of a piece) can be linked to a "work" — which represents the underlying musical composition. For a classical piece like a symphony, individual movements are separate works, but they all belong to a larger parent work (the symphony as a whole). Before this PR, beets could store the `mb_workid` for a recording but had no ability to climb up the hierarchy and retrieve the top-level composition's details.

The previous behaviour was that metadata fields like `work`, `composer`, `work_date` were either absent or had to be filled in manually. There was no automated way to retrieve the parent composition's information.

The new behaviour introduced by this PR is that when the `parentwork` plugin is enabled, beets queries the MusicBrainz API using the stored `mb_workid`. It then traverses the work-to-parent-work relationships upward through the hierarchy until it reaches the root composition. The plugin collects and stores several new metadata fields: the name of the parent work, the composer(s) involved, the composition date, and an ordered list of work IDs representing the full hierarchy path. Configuration options include `force` (to re-fetch even when data is already present) and `use_cache` (to avoid repeated API calls during a single session).

---

## 3.1.3 Acceptance Criteria

✓ When a track has a valid `mb_workid` stored and the `parentwork` plugin is enabled, the plugin should successfully query the MusicBrainz API and return work relationship data without errors.

✓ When a work has a parent work relationship in MusicBrainz, the plugin should recursively traverse the hierarchy and store the **top-level** parent work's title in the `work` field — not an intermediate work's title.

✓ When a work has no parent (it is already the root composition), the plugin should store that work's own title in the `work` field and stop traversal without throwing an exception.

✓ When the `force` configuration option is set to `False` (the default), the plugin should skip re-fetching data for items that already have a non-empty `work` field.

✓ When the `force` configuration option is set to `True`, the plugin should re-fetch and overwrite existing `work`, `composer`, `work_date`, and `mb_workhierarchy_ids` fields, even if they are already populated.

✓ When `use_cache` is enabled, the plugin should not make a duplicate MusicBrainz API call for the same work ID within a single beets session — the first result should be reused from an in-memory cache.

✓ When a track has no `mb_workid` stored (e.g., it was not imported via MusicBrainz), the plugin should gracefully skip that item without raising an error or crashing the beets process.

✓ The new metadata fields (`work`, `composer`, `work_date`, `mb_workhierarchy_ids`) should be correctly written to the item in beets' library database after the plugin runs.

---

## 3.1.4 Edge Cases

**Edge Case 1: Circular or deeply nested work hierarchies**  
MusicBrainz data is community-contributed and can occasionally have data quality issues. A work might theoretically reference itself or form a loop in its parent relationships. The plugin must include a guard (e.g., a visited set or maximum depth limit) to prevent infinite recursion if it encounters a circular parent-child relationship.

**Edge Case 2: MusicBrainz API rate limiting or network failure**  
The MusicBrainz API enforces rate limits (1 request/second for anonymous access). When running the plugin against a large library, rapid successive API calls may result in HTTP 503 or 429 responses. The plugin should handle these responses gracefully — either by implementing a retry with backoff or by logging a warning and skipping the item — rather than crashing the entire import/write operation.

**Edge Case 3: Works with multiple composers or ambiguous composer credits**  
A parent work may have multiple composer credits (e.g., co-compositions, or cases where arranger/orchestrator credits exist alongside the original composer). The plugin must decide how to handle this — whether to concatenate multiple names, pick the first, or store all of them — and must not crash when the MusicBrainz response contains a list of composer relations rather than a single one.

**Edge Case 4: Missing or malformed `mb_workid` field**  
The `mb_workid` field might be present but contain an empty string, `None`, or a malformed UUID (not a valid MusicBrainz identifier). The plugin should validate the field before making any API call and skip gracefully if the ID is not a valid UUID format, logging an appropriate warning message.

---

## 3.1.5 Initial Prompt

```
You are implementing a new plugin for the beets music library manager (https://github.com/beetbox/beets), a Python-based CLI tool for music collection management. Beets uses a plugin architecture where all plugins live in the `beetsplug/` directory as Python modules and are registered via entry points.

## Task

Implement the `parentwork` plugin as described in PR #3279 (https://github.com/beetbox/beets/pull/3279). This plugin enriches music metadata for classical and jazz recordings by querying the MusicBrainz API to find the top-level parent work of a recording and storing that information as new metadata fields.

## What the plugin must do

1. Hook into beets' `write` event so it runs when item metadata is being written to disk.
2. Read the `mb_workid` field from the beets item. If this field is absent, empty, or not a valid UUID, skip the item silently with a debug log message.
3. Query the MusicBrainz API (using the `musicbrainzngs` library already available in beets) to retrieve the work associated with the `mb_workid`.
4. Traverse the work's "parts of" / parent-work relationships recursively upward until the root composition is reached (no further parent exists).
5. Store the following fields on the item:
   - `work`: title of the top-level parent work
   - `composer`: name(s) of the composer(s) of the parent work
   - `work_date`: earliest composition date found in the hierarchy
   - `mb_workhierarchy_ids`: pipe-separated string of all work IDs traversed, from the original to the root

## Configuration options to support

- `force` (boolean, default: False): if False, skip items that already have a non-empty `work` field; if True, always re-fetch and overwrite
- `use_cache` (boolean, default: True): if True, cache MusicBrainz API results in memory per session to avoid duplicate requests

## Acceptance criteria to satisfy

- ✓ A track with a valid `mb_workid` and an existing parent-work relationship in MusicBrainz gets its `work` field populated with the root composition title
- ✓ A track whose work has no parent (it is already the root) gets the work's own title stored without error
- ✓ Items without `mb_workid` are skipped without exceptions
- ✓ `force=False` preserves existing `work` values; `force=True` overwrites them
- ✓ `use_cache=True` avoids redundant API calls within a session

## Edge cases to handle

1. **Circular hierarchy protection:** Keep a set of already-visited work IDs during traversal; if a work ID is encountered again, break the loop to prevent infinite recursion.
2. **API rate limiting / network errors:** Wrap all MusicBrainz API calls in try/except. On network error or rate-limit response, log a warning and skip the item gracefully rather than crashing.
3. **Multiple composer credits:** If MusicBrainz returns multiple composer relations, join the composer names with a separator (e.g., ", ") rather than failing.
4. **Malformed `mb_workid`:** Validate that the stored value matches UUID format before making any API request.

## File structure

- Create: `beetsplug/parentwork.py`
- Create: `docs/plugins/parentwork.rst` (basic documentation)
- Update: `docs/plugins/index.rst` to include `parentwork` in the plugin list

## Testing requirements

Write unit tests in `test/test_parentwork.py` that:
- Mock the MusicBrainz API responses (do not make real network calls in tests)
- Test the traversal logic with a 2-level hierarchy (recording → movement → symphony)
- Test the skip behaviour when `mb_workid` is missing
- Test the `force` flag behaviour
- Test circular reference protection (mock a response where work A's parent is work B and work B's parent is work A)

## Reference

Follow the structure of an existing simple beets plugin (e.g., `beetsplug/zero.py` or `beetsplug/missing.py`) for plugin class setup, config declaration, and event listener registration. Use `musicbrainzngs.get_work_by_id()` for API calls with the `includes=["work-rels", "artist-rels"]` parameter.
```
