# Esipher build template

A GitHub Actions workflow that turns raw AI-site-builder source into a
built, Esipher-ready ZIP — for users who don't have Node.js installed
and don't want to use the command line.

## Prefer video? Watch the full walkthrough: https://youtu.be/6-ne9-7Wgt4

It detects npm/yarn/pnpm/bun automatically, installs dependencies, runs
the project's build script, and hands back the output (`dist/`,
`build/`, `out/`, `.output/public/`, or `static-export/` — whichever
exists) as a downloadable artifact. No server of yours is involved:
GitHub's own runners do the work, for free.

It mirrors the same build intelligence Esipher's own "Build it now"
button uses (`Project_Builder`), so a project this workflow can build,
Esipher can too, and vice versa:
- **Lovable cloud-hosted images** — if the source has Lovable's
  `*.asset.json` descriptors (under `src/assets/`), the workflow reads
  the project ID from one and sets `ASSET_ORIGIN` before building, so
  the project's own asset-download step (if it has one) can actually
  reach those images instead of silently failing every one.
- **Export-static fallback** — some frameworks (TanStack Start among
  them) need a second script after the plain `build` to pre-render
  every route to real HTML. If the plain build's output isn't
  recognizable, the workflow tries `export:static`/`export-static`/
  `static:export` next, same as Esipher itself does.
- **Nitro output bridging** — some of those export scripts hardcode
  expecting their output at `dist/server`/`dist/client`, out of sync
  with what their own `vite.config` (via Nitro) actually produces
  (`.output/server`/`.output/public`). The workflow bridges the two
  before retrying, never overwriting a `dist/` that's already there.
- **Always builds fresh** — `static-export/`, `dist/`, `build/`, `out/`,
  and `.output/` are all removed right after checkout, before anything
  runs. Some exports bundle a `static-export/` folder together with the
  raw source it was built from; left in place, that's indistinguishable
  from genuine output to the step that looks for a finished build, so
  it would get used as-is without this workflow ever actually running
  a build — potentially stale or missing files the original export's
  own asset-download step never got to. Clearing all of these up front
  guarantees the artifact you download only ever came from this run.

## One-time setup (done — kept here for reference)

Live at **https://github.com/adebayomoses/Esipher-Build**, linked from the
admin page's own "Uploading raw source code instead of a build? Read this
first." help text (see `admin/class-admin-page.php`) and from `readme.txt`'s
FAQ. Steps below are what stood it up, kept for maintaining it or standing
up a second one later:

1. Create a new, empty repository on GitHub (e.g. `esipher-build-template`).
2. Copy this folder's contents — just the `.github/workflows/build.yml`
   file — into that repo's root, so the path is
   `.github/workflows/build.yml` in the new repo.
3. Commit and push.
4. In that repo's **Settings**, check **"Template repository"**. This
   is what lets end users click "Use this template" and get their own
   copy, workflow included, with zero git knowledge.

## What an end user does with it

No account, install, or command line beyond a browser:

1. Open your template repo's page and click **"Use this template" →
   "Create a new repository"**. Any name is fine. This gives them
   their own copy, workflow file included.
2. In their new repo, click **Add file → Upload files**, then drag in
   the **raw `.zip` file exactly as their AI builder exported it** —
   no need to unzip it first. (GitHub doesn't extract zips on upload,
   so it just gets committed as one file — the workflow itself detects
   and extracts it before building. If they'd rather drag in loose
   files instead, that still works too; the workflow only extracts
   when it finds a `.zip` at the root.)
3. Commit — this push triggers the workflow automatically.
4. Wait about a minute, then open the **Actions** tab, click the
   run that just completed, and scroll to **Artifacts** at the bottom.
5. Download `esipher-build.zip` and upload *that* to the Esipher
   plugin.

## Known limitations

- Requires a (free) GitHub account.
- GitHub's web uploader blocks files over 100 MB (and warns above
  25 MB) — fine for typical site-export source, but a very large
  export's zip could hit that ceiling. If so, extracting locally and
  dragging in the loose files (skipping large asset folders like
  stock video) is the workaround.
- If more than one `.zip` happens to be sitting at the repo root, the
  workflow just picks whichever one `find` lists first — not
  something a normal single-export upload would ever trigger, but
  worth knowing if a repo gets reused for multiple exports.
- If the project's `package.json` has no `build` script at all, the
  "Build" step fails outright — the run's log shows the real error
  from npm/yarn/pnpm/bun. (A `build` script that succeeds but needs a
  *second* export-static-style script afterward is handled
  automatically — see above; it's specifically *no build script at
  all* that isn't recoverable.)
- Private repos on a free GitHub plan get 2,000 Actions minutes/month;
  public repos get unlimited minutes. One build here takes roughly
  1–3 minutes, so this isn't a practical constraint for occasional use.
- This workflow doesn't validate the output's *structure* the way the
  planned "Dist Packager" tool would (root-absolute asset paths, stray
  `node_modules/` left in, etc.) — it just runs the build and hands
  back whatever came out. Esipher's own upload step still does its
  usual validation.
