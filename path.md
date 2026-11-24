# path.md — Reference layer patch (OpenFill for PhotoCraft)

OpenFill for PhotoCraft is a companion app plus a small Wasm plug-in. PhotoCraft plug-ins are
sandboxed (no files/network/processes), so AI runs in the companion over PhotoCraft's control
channel. PhotoCraft's install method and OS are irrelevant; keep the code OS-neutral.
`openfill/` is shared with OpenFill for GIMP: no GIMP imports, no PhotoCraft assumptions.
`openfill_photocraft/` is the PhotoCraft side.

`ROOT` is the folder that contains this file. The package to patch is the one the setup put
next to it: `ROOT/OpenFill/openfill-photocraft-<ver>` (the versioned folder). Patch that copy
and no other OpenFill installation.

## Task
Add a **Reference** select right after **Model** (Fill and Remove only). The chosen layer is
sent to FLUX.2 as an extra reference image (Krita-style reference control layer). It must not be
composited into the scene while the job runs, and its visibility must be restored afterwards.
Default is `None` (stock behavior).

## Canonical decisions (follow exactly; do not improvise)
1. **Layers listed:** picture layers only. Skip `kind` in ("group", "folder"). A store that
   resolves to a group is ignored with a log line.
2. **Value:** store the exact layer name when it is unique and not numeric; otherwise
   store `id:<n>`. Resolution accepts `id:<n>`, exact name, or bare id.
3. **UI list:** `None` is always the first choice in every refresh. Duplicate names are shown
   as `Name (ID n)`. A stored spec that is not in the current document becomes `None`; never
   add an unknown label to the list. Drop that document's entry from the map.
4. **Tab gating:** the row is packed only for `fill` and `remove`; hidden otherwise (Expand,
   Cutout, Upscale, Create, custom workflow). When packed or re-packed, give it the same top
   gap as the Model row (`pady=ui.px((8, 0))`); `pack_forget` drops the old padding.
5. **Refresh and memory:** off the UI thread, keyed by (tab, document identity), at most every
   2 s, skipped while `service.busy()` or while the settings view is open. The UI thread only
   renders. Keep a session map document → spec, keyed by the document's tab index and name (or
   its path when the bridge reports it); cap it at 32 documents. When the document changes, put
   the live value back under the old document, set `settings.reference_layer` to the new
   document's stored spec (empty when it has none), show that at once, and refresh. A choice is
   never applied to another document. Switching between Fill and Remove is not a document
   change and must not clear it. A reply for a document that is no longer active is discarded.
6. **Session state:** never save the choice. `Settings.reference_layer` is not persisted
   (`field(default="", metadata={"persist": False})`); `App.reload_settings` keeps the live
   value so saving other settings does not drop it.
7. **Scope:** the reference is used only when `model.effective_template == "flux2_edit"`; every
   other model ignores it with a log line (no export, no upload, no hide).
8. **Scaling:** read the reference PNG header (`png.read_header`) in the builder; fit it inside
   the work box preserving aspect; floor both sides to a multiple of 16.
9. **Failure policy:** missing layer, group, export failure, hide failure, upload failure —
   log and run the job without the reference. Never raise, never fail the job.
10. **Visibility:** capture every layer's visibility before any change; restore all in
    `finally` **immediately after the scene and mask are captured** (before the generation
    request), not after the job; log restore failures instead of raising. Do **not** touch
    history/rewind.
11. **Snapshot:** reset `image._snapshot` after every visibility change.
12. **One reference** per job. No multi-reference list.
13. **Files:** change exactly the eight files in "Changes" below. No `service.py`, no new
    modules, no other package files. Helper functions inside those files are fine.
14. **Logging:** the bridge returns a result or raises; `generative_fill` logs. Do not add
    logging plumbing to `service.py` or the bridge.

## Behavior
- Choice lives only while the plug-in runs, separately for each open document; switching back
  restores it; nothing survives a restart.
- After success, error or cancel: visibility and selection exactly as before; one new layer,
  one undo step for the result.
- The reference never enters the scene composite.

## Changes (anchor on symbols, not lines)
1. `Settings` — `reference_layer: str = field(default="", metadata={"persist": False})`
   (a live value only; it never reaches `config.json`).
2. `EDITABLE` — `"reference_layer": str`.
3. `InpaintRequest` — `reference_png: bytes = b""`.
4. `ComfyBackend.inpaint` — upload as `sb-<job>-reference.png` inside its own try/except; set
   `request.reference_ref`; clear it per request.
5. `build_flux2_edit` — if `reference_ref`, add `LoadImage → ImageScale (rule 8) → VAEEncode →
   ReferenceLatent` to positive and negative, then feed `CFGGuider`. Without it, stock graph.
6. `generative_fill` — resolve setting, export via bridge, hide before scene/mask capture,
   restore immediately after the capture in `finally`, reset snapshot, pass `reference_png`.
   `App.reload_settings` — keep `reference_layer` across reloads (session state).
7. `PhotoCraftBridge` — `export_layer_png(image, spec, max_side=1024)` (resolve per rule 2,
   isolate layer + parent groups, snapshot, crop bounds/bbox, restore visibility) and
   `set_layer_visible`.
8. `bar.py` — `ui.Select` after the model row with the same 8 px top gap (rule 4); rules 3–6.

## Invariants
- Document: new layer, one undo step for the result, visibility/selection restored.
- `None` ⇒ stock graph and behavior, unchanged.
- Reference failure never fails the job.
- No control/ComfyUI calls on the UI thread; no secrets in logs; nothing outside the profile.
- Additive, idempotent; backup and deterministic revert exist.

## Rules
- Symbols, not line numbers; fail loudly on missing anchors.
- Probe capabilities, not versions (`getattr(req, "reference_ref", None)`, `caps.*`,
  `getattr(settings, "reference_layer", "")`).
- Edit only the FLUX.2 builder. Do not touch `build_fill`, `insert_results`,
  `show_variation`, the model catalog, supervisors or ports.
- If upstream already supports references (field, node, setting), do nothing and log
  `already supported upstream`.

## Tools
Use the private runtime installed by `setup.md` for every check:
- Windows: `ROOT/python/python.exe`
- Linux/macOS: `ROOT/python/bin/python3`

Fall back to `python3` only when the private runtime is absent.

## Verify
- `py_compile` + import; build `build_flux2_edit` with and without `reference_ref`: stock node
  set identical, reference path wired to both conditionings, one `ImageScale` with multiples of
  16, no duplicate ids.
- Changed files are exactly the eight listed; `service.py` and all other files untouched.
- Restart the plug-in: `config.json` has no `reference_layer`; the select shows `None`.
- Two documents: select a layer in A, switch to B and back — A's choice is restored; a value
  from A is never applied to B; a missing layer becomes `None` with no raw entry.
- After switching tabs, the Reference row keeps the same 8 px gap above it as the Model row.
- One end-to-end run on a **copy** of a document: new layer, restored visibility, reference log
  line; with `None` the graph contains no reference nodes.

## Where to patch
The target is the OpenFill package, not PhotoCraft. In this setup it is the versioned folder
next to this file:
- `ROOT/OpenFill/openfill-photocraft-<ver>`

Use an installed copy only when that folder is absent and the user explicitly asks for it:
- Windows: `%LOCALAPPDATA%\OpenFill for PhotoCraft\app`
- Linux: `~/.local/share/openfill-photocraft/app`

An OpenFill update replaces the package: re-apply the patch to the new one. `path.md` itself
does not change.

## If the patch fails
- Restore `<pkg>.BACKUP` first, re-run `py_compile` and the graph check on the restored package,
  then stop.
- Do not repair a broken package in place; keep the failed copy or diff for diagnosis.
- Never delete or rewrite the user's settings, profile or models.

## Rollback
Copy the package to `<pkg>.BACKUP` before editing; restore it to revert. Ship a short note
outside the package, so the changed-file set stays exactly eight.
