---
name: update-modpack-mods
description: 'Look up the newest Thunderstore versions for mods in this Valheim modpack and update modpack/manifest.json and modpack/CHANGELOG.md accordingly. Use when asked to update mods, check for mod updates, bump mod versions, or refresh the modpack for a new Valheim/mod release.'
---

# Update Modpack Mods

## Procedure

1. Read `modpack/manifest.json` to get the current `Namespace-PackageName-Version` list.
2. For each dependency, look up the latest version on Thunderstore:
   - Do **not** use `curl`/terminal for `thunderstore.io` - it is blocked by a Cloudflare JS
     challenge (returns HTML challenge page instead of real content), even with a browser
     User-Agent header. Don't retry curl with different headers, it won't work.
   - Use the `fetch_webpage` tool instead.
   - The REST API (`/api/v1/package/...`) 404s via fetch_webpage - use the human package page:
     `https://thunderstore.io/c/valheim/p/<Namespace>/<PackageName>/`
   - The version is embedded in the install/download links on that page, e.g.
     `ror2mm://v1/install/thunderstore.io/<Namespace>/<PackageName>/<VERSION>/` or
     `https://thunderstore.io/package/download/<Namespace>/<PackageName>/<VERSION>/`.
     Ask fetch_webpage specifically for "the version number in the install/download link" to
     avoid pulling in the huge amount of boilerplate/description/ad content each page has.
   - Call `fetch_webpage` with only ~3 URLs per call (2 parallel calls of 3 max at a time).
     Requesting more (e.g. 5+ per call, or 10+ across parallel calls) has triggered HTTP 429
     rate-limiting/failures in the past - go smaller and slower rather than bigger and faster.
   - The new package page occasionally returns a generic `Uh oh! Something went wrong` error
     (cyberstorm UI bug, unrelated to rate limiting - retrying the same URL keeps failing). When
     that happens, fall back to the legacy site instead of retrying:
     `https://old.thunderstore.io/c/valheim/p/<Namespace>/<PackageName>/` - it reliably renders and
     shows the version in a `Dependency string` line as well as the install/download links.
3. Compare each latest version to the manifest's current version. Only touch mods that changed.
4. Update `modpack/manifest.json`:
   - Edit only the `dependencies` array entries that changed (`Namespace-PackageName-OldVersion` ->
     `Namespace-PackageName-NewVersion`).
   - Do **NOT** change the top-level `"version_number"` field - it always stays `"1.0.0"` in the
     committed file (verified across all past release tags v1.0.0-v1.8.0 via
     `git show <tag>:modpack/manifest.json`). The real publish version is injected at release time
     by `.github/workflows/publish.yaml` (triggered on GitHub Release published), not stored here.
5. Update `modpack/CHANGELOG.md`:
   - Add a new entry at the **top** (right after the `# Changelog` header / before the previous
     newest entry), in this exact style:
     ```
     ## [X.Y.Z] - YYYY-MM-DD

     ### UPDATE

     - Namespace-PackageName-NewVersion
     - ...

     ---
     ```
   - Use `### ADD` / `### REMOVE` sections too if mods were added/removed (see file for examples).
   - Bump the version as a minor version bump from the last changelog entry (e.g. 1.8.0 -> 1.9.0).
   - Use today's date in `YYYY-MM-DD` format.
   - Entries in the file are separated by a `---` line; keep that convention for the new entry too.
6. Summarize the mod version changes made (old -> new) to the user, and note any mods that were
   already up to date.
