# mu-dsh-files

A [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (dsh)
plugin for the web UI:

* **A Files tab that follows the disk.** It takes over dsh's own Files tab,
  so dsh's guide card and Workspace files shortcut open it. It is a tree of
  the session's working directory: expand folders inline, open files, go up
  a level, reload. While it is visible, every folder shown open is watched
  and listed again shortly after it changes.
* **Folder links.** A link to a folder in a reply (or anywhere dsh opens
  files) opens a Files tab rooted at that folder instead of failing in the
  file preview. A link to the working directory opens the Files page.
* **Downloads.** There are download buttons on dsh's file preview, on its
  "cannot preview" state, and on every row and header of a Files tab.
  Folders download as ZIP archives.

It is part of [mu-dsh](https://github.com/SleepyMug/mu-dsh), which installs it
into every user's dsh as a git submodule under `plugins/`. It depends on
nothing of mu-dsh's, but it is only built and tested the way mu-dsh uses it,
against the dsh version mu-dsh pins (currently `0.2.1-rc.5`).

## How it works

The plugin has two halves, both in `lib/` (built output, committed so that
installing needs no build step):

* `lib/index.js`, the **host half**, adds three exact routes to dsh's `/api`:
  * `mu-dsh-files.entry`: what a path is, and a folder's entries.
  * `mu-dsh-files.download`: a file, or a folder as a streamed ZIP. `HEAD`
    runs every check without sending a body.
  * `mu-dsh-files.watch`: watch streams, described below.

  dsh authenticates these routes like its own (only the profile's owner
  reaches them). They go through dsh's filesystem service, so they read
  exactly what the user's dsh may.
* `lib/client.js`, the **browser half**, is loaded by dsh's web client.

**Taking over the Files tab.** dsh's right sidebar allows one `builtin` and
one `extension` registration per tab kind, and uses the extension while it
is registered. The plugin registers kind `files` as an extension, so
everything that opens dsh's Files page opens this one. If the plugin fails
or is turned off, dsh's own tab resumes.

**Folder links.** Every file address goes to one tab type, chosen before
anything can know whether the path is a file or a folder. The plugin's type
claims all of them, ranked above dsh's viewers, and each such tab first asks
the host what its path is:
* A folder stays, as a tree.
* The working directory becomes the Files page.
* Anything else reopens in the tab's place with the viewer dsh would
  otherwise have used, keeping the line a link points at.

The cost is one small request before every file opens.

**Watching.** A browser page holds one long-lived stream: newline-delimited
JSON events, with a ping every 20 s. It names the folders to watch with
small `POST`s beside the stream, so watching many folders costs one
connection. The host keeps each watch up:
* it retries a failed watch with backoff;
* it watches a path again when it becomes, or stops being, a folder;
* it reports at most one change per folder per second.

The browser watches only folders shown open in a visible tab. If the stream
is lost, it opens another and asks for the same folders again. Meanwhile a
dot on the reload button says changes may be missing. The event types are
documented in `src/wire.ts`.

**Downloads.** The button asks first (`HEAD`), so a refusal shows next to it,
naming the path. A refusal means dsh may not read it, or the folder is over
65,535 entries or 2 GiB. Then the browser's download manager streams the file
to disk. Archives are walked in full before the first byte, so a refusal is
an HTTP status rather than a broken archive.

**Failing safely.** Any client plugin that fails to load stops dsh's whole
web UI. This one cannot fail its own load. Its parts wait for the dsh
services they use, in a scope of their own. If a part fails, it is logged
and undone, and dsh starts without it.

To turn the plugin off for one user, add these lines to their profile patch:

```yaml
- id: mu-dsh-files
  disabled: true
```

## Layout

| Path | What |
|---|---|
| `src/wire.ts` | route paths and what they carry; shared by both halves |
| `src/index.ts` | host half: registers the routes |
| `src/entry.ts`, `src/download.ts`, `src/watch.ts` | the routes |
| `src/files.ts`, `src/http.ts` | the filesystem interface the routes use, and response helpers |
| `src/client/index.ts` | browser half: registers the tab type, its body and title, and the download buttons |
| `src/client/tree.ts`, `src/client/watch.ts` | a tab's state, and the page's watch stream |
| `src/client/Files.tsx`, `src/client/Download.tsx` | what they draw |
| `src/client/api.ts`, `src/client/paths.ts` | the routes as the browser calls them; addresses and paths |
| `build.mjs` | builds both halves into `lib/` |
| `test/` | Node tests for the routes, the watches and the tab state |

## Development

Requires Node ≥ 22 and pnpm.

```sh
pnpm install
pnpm run typecheck
pnpm test
pnpm run build          # writes lib/; commit it with the sources
node build.mjs --check  # fails if lib/ is out of date
```

dsh's packages are dev dependencies, used for types only. The host half
bundles its one runtime dependency (fflate). The browser half requires only
what dsh's web shell provides (React and dsh's UI libraries). The build fails
on any other import.
