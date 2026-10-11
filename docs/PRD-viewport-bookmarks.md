# PRD — Viewport Bookmarks

**Feature:** Save, recall, and navigate between bookmarked zoomed regions of an image.
**Target:** qView 7.x (Qt 6 / Qt 5), C++14.
**Status:** Revised after source audit — §11 decisions resolved, ready for maintainer review.
**Author:** Francesco Lardieri.

> **Revision note (2026-10-11).** Audited against `main` @ `c5eca1c`. Three changes from the
> first draft: (a) §2's claim that qView has no mirror/flip state was **wrong** — it exists in the
> view transform, so mirror/flip are now captured (§4, §6); (b) the four open questions in §11 are
> resolved with rationale, including the macOS keybinding question verified against Qt 6.11.1;
> (c) a pre-existing, unrelated shortcut defect found during the audit is recorded in §14.

---

## 1. Summary

Add the ability to **bookmark a viewport** — the current image plus its zoom level, pan
position, and orientation — and later jump back to that exact composition. Users can save any
number of bookmarks, cycle between them with the keyboard, and manage them from a menu (and,
optionally, a sidebar strip). This brings to qView the "bookmark zoomed regions and navigate
between them" capability proven in the sibling project *SleekView*.

The primary use case is detailed inspection of large images (maps, scans, artwork, diffs,
photography): a user zooms into one region of interest, bookmarks it, zooms into another,
bookmarks that, and then flips between the saved regions instantly instead of re-finding them
by hand.

## 2. Goals / Non-Goals

### Goals
- Capture the current viewport (file + zoom + pan + rotation + mirror/flip) as a
  named-by-default bookmark.
- Restore a bookmark exactly, robust to window resizes between save and restore.
- Navigate the bookmark list with the keyboard (next / previous, wrapping).
- Persist bookmarks across sessions using qView's existing settings mechanism.
- Expose all operations through the existing menu/action/shortcut architecture, fully
  translatable.
- Keep the feature self-contained and behind sensible defaults so it never disrupts users who
  don't use it.

### Non-Goals (v1)
- Thumbnail-cropped previews of each bookmark (deferred to v2; see §10).
- Drag-and-drop reordering (deferred to v2).
- Cross-device sync or export/import of bookmark files.
- Bookmarking animated-frame positions within a GIF/movie (see §11 Q4).

## 3. Reference Behavior (SleekView parity map)

SleekView is the design reference. This table maps each SleekView behavior to the qView v1
decision so reviewers can see exactly what is kept, changed, or deferred.

| SleekView behavior | qView v1 decision |
|---|---|
| Bookmark = `{fileName, zoom, panX, panY, viewportW, viewportH, createdAt, orientation}` | Same shape; `orientation` decomposes into `rotation` (int degrees) + `mirrored`/`flipped` (bool), matching how qView stores it |
| Folder-scoped JSON sidecar `.sleekview.json` | **Changed** → `QSettings` store (qView never writes into the user's image folders; matches the recents-list pattern) |
| Flat list navigation, Arrow Up/Down, wraps | Same semantics; `Ctrl+Up` / `Ctrl+Down` defaults (§7), wrap preserved |
| Right-edge thumbnail strip with crop previews | **Deferred to v2**; v1 ships a Bookmarks **menu**. Strip can reuse the merged sidebar (PR #838) |
| Delete-on-hover "×", Clear-all footer | v1: delete + clear-all via menu; hover UI lands with the strip in v2 |
| Drag-and-drop reorder | **Deferred to v2** |
| Orphan bookmarks (missing file kept, flagged) | Kept; orphan rules in §8 |
| Restore order: orientation → zoom (clamped) → pan (clamped) | Same restoration order, adapted to qView's transform model (§6) |
| Bookmark scope: whole folder | **Changed** → global flat list (§11 Q1 for rationale) |

## 4. Data Model

A bookmark is a value struct, serialized via `QVariant` exactly like `ActionManager::SRecent`
([actionmanager.h:55-73](../src/actionmanager.h#L55)).

```cpp
struct Bookmark
{
    QString absoluteFilePath;   // identifies the image; also the orphan key
    QString fileName;           // cached display label
    qreal   zoom;               // QVGraphicsView::currentScale at save time
    QPointF sceneCenter;        // image-space point centered in the viewport (resize-robust)
    int     rotation;           // QVImageCore::getCurrentRotation() (degrees, 0/90/180/270)
    bool    mirrored;           // transform().m11() < 0
    bool    flipped;            // transform().m22() < 0
    QSize   viewportSize;       // viewport px at save time (for clamping math on restore)
    qint64  createdAtEpochMs;   // creation timestamp (sortable, locale-free)
};
```

### Why `sceneCenter` instead of raw scroll positions
qView pans by moving scrollbars ([qvgraphicsview.cpp:272](../src/qvgraphicsview.cpp#L272)) and
recenters via `centerOn(QPointF)` ([qvgraphicsview.cpp:735](../src/qvgraphicsview.cpp#L735)).
Storing the **scene-space center point** (the image coordinate under the middle of the
viewport) rather than absolute scrollbar values makes restoration independent of window size —
directly mirroring SleekView's "apply pan, then clamp against current viewport" rule (PRD §9.5).
`viewportSize` is retained only as metadata for clamping decisions and future thumbnail cropping.

### Why `mirrored` / `flipped` exist
The first draft asserted qView models rotation only. That was wrong. Mirror and flip are real,
default-bound user actions (`F` and `Ctrl+F`): `MainWindow::mirror()` is
`graphicsView->scale(-1, 1)` and `flip()` is `scale(1, -1)`
([mainwindow.cpp:1019-1029](../src/mainwindow.cpp#L1019)). The state lives in the **view
transform**, not `QVImageCore`, which is why it is invisible from the image-core API. qView
itself reads it back by sign-testing the transform — see `scaleExpensively` and `makeUnscaled`
([qvgraphicsview.cpp:409-416](../src/qvgraphicsview.cpp#L409),
[:460-467](../src/qvgraphicsview.cpp#L460)), which detect it, undo it, re-apply it around every
rescale. Capturing rotation alone would silently restore a mirrored bookmark un-mirrored. Both
flags are one comparison to read and one `scale()` call to restore, so this is cheap parity with
SleekView's single `orientation` field.

## 5. Persistence

Mirror the recents-list implementation in
[actionmanager.cpp:330-355](../src/actionmanager.cpp#L330):

- Store under a dedicated `QSettings` group, e.g. `bookmarks/`.
- Serialize the bookmark list to a `QVariantList` (each bookmark → `QVariantMap` or ordered
  `QStringList`/`QVariantList`, as recents does).
- Debounce writes with a `QTimer` (recents uses `recentsSaveTimer`, 500 ms single-shot) to avoid
  thrashing on rapid save/delete.
- Add a `SaveBookmarks` toggle to `SettingsManager::Setting`
  ([settingsmanager.h:17](../src/settingsmanager.h#L17)), defaulting **on**, paralleling
  `SaveRecents` — one enum entry plus one registration line next to
  `{ Setting::SaveRecents, true, "saverecents" }`
  ([settingsmanager.cpp:57](../src/settingsmanager.cpp#L57)).

**Scope key:** one global flat list under `bookmarks/bookmarkList`, matching how recents stores a
single `recents/recentFiles` list. No folder keying — see §11 Q1. Unlike recents there is no
length cap: bookmarks are explicit user acts, not history, so silently evicting the oldest would
be data loss. If a cap is wanted for settings-file size, make it generous (≥100) and documented.

## 6. Capture & Restore

### 6.1 Capturing (Save Bookmark)
On trigger, read live viewport state from `QVGraphicsView`:
- `zoom` ← `currentScale`
- `sceneCenter` ← scene point currently at the viewport center (map viewport center through the
  inverse of `absoluteTransform`)
- `rotation` ← `imageCore.getCurrentRotation()`
- `mirrored` / `flipped` ← `transform().m11() < 0` / `transform().m22() < 0`
- `absoluteFilePath` / `fileName` ← `getCurrentFileDetails().fileInfo`
- `viewportSize` ← `viewport()->size()`

**Required additions:** `QVGraphicsView` currently exposes none of `currentScale`,
`absoluteTransform`, or a scene-center getter (all `private`,
[qvgraphicsview.h:121-138](../src/qvgraphicsview.h#L121)). Add:
```cpp
struct ViewportState {
    qreal zoom;
    QPointF sceneCenter;
    int rotation;
    bool mirrored;
    bool flipped;
    QSize viewportSize;
};
ViewportState getViewportState() const;
void applyViewportState(const ViewportState &state);
```
Keeping capture/restore math inside `QVGraphicsView` (where the transform lives) avoids leaking
private members and keeps the bookmark manager UI-only.

### 6.2 Restoring
Adapt SleekView's ordered restore (PRD §9.5) to qView:
1. If `absoluteFilePath` differs from the loaded file, load it via `loadFile()` **without**
   resizing the window, so the restored composition slots into the current canvas.
2. Apply `rotation` first (so subsequent fit/clamp math uses rotated bounds), via the existing
   `rotateImage` path.
3. Apply `mirrored` / `flipped` by `scale(-1, 1)` / `scale(1, -1)` **only if** the current
   transform's sign differs from the target — the flags are a toggle, not an absolute set, so
   applying them unconditionally would invert an already-mirrored view. Do this before zoom, for
   the same reason rotation comes first: `scaleExpensively` and `makeUnscaled` fold the
   mirror/flip signs into every rescale.
4. Apply `zoom`, clamped to qView's existing scale limits
   ([qvgraphicsview.cpp:371](../src/qvgraphicsview.cpp#L371), `0.01 … 500`).
5. `centerOn(sceneCenter)`, then let qView's existing scroll clamping keep it in-bounds. This
   handles viewport-size changes since save time.

Note that `MainWindow::mirror()`/`flip()` and `rotateImage()` each call `resetZoom()` afterwards,
so restore must not route through those wrappers — it operates on the view directly, inside
`applyViewportState`, or the zoom applied in step 4 would be discarded.

## 7. Navigation & Actions

All actions are registered in `ActionManager::initializeActionLibrary` and surfaced via
`buildViewMenu` ([actionmanager.h:104](../src/actionmanager.h#L104)), dispatched through
`actionTriggered` ([actionmanager.h:126](../src/actionmanager.h#L126)) — the same path every
existing command uses, so they inherit cloning, menubar/context-menu placement, and the shortcut
editor automatically.

| Action key | Label | Behavior | Default shortcut |
|---|---|---|---|
| `saveBookmark` | Save Bookmark | Capture current viewport; append; auto-select it | `Ctrl+D` (⌘D on macOS) |
| `nextBookmark` | Next Bookmark | Cycle to next in list (wraps); restore | `Ctrl+Down` |
| `previousBookmark` | Previous Bookmark | Cycle to previous (wraps); restore | `Ctrl+Up` |
| `clearBookmarks` | Clear Bookmarks… | Confirm dialog, then empty the list | — |

Add a **Bookmarks** submenu (built like `buildRecentsMenu`,
[actionmanager.cpp:300](../src/actionmanager.cpp#L300)) listing saved bookmarks as
`"<fileName> — <zoom>×"` entries; clicking one restores it. Include the Clear action at the
bottom, gated on a non-empty list, exactly as recents does.

**Navigation index reset:** when the user changes images by normal navigation
(`goToFile`, [qvgraphicsview.cpp:552](../src/qvgraphicsview.cpp#L552)), reset the current
bookmark selection index to "none" — matching SleekView's `currentBookmarkFlatIndex = -1` on
Left/Right.

Shortcuts are registered with defaults in `ShortcutManager::initializeShortcutsList`
([shortcutmanager.cpp:41](../src/shortcutmanager.cpp#L41)) and remain user-rebindable through the
existing shortcut dialog — no new settings UI required for keybindings. Registering there is
**mandatory, not optional**: the shortcut editor iterates `shortcutsList`, so an action absent
from it cannot be rebound by the user at all (§14 is a live example of that failure).

## 8. Orphan Bookmarks

Adopt SleekView's orphan policy (PRD §9.7) — never silently destroy a bookmark for a
temporarily missing file:

| Situation | Behavior |
|---|---|
| File missing at load | Keep the bookmark in memory and storage. |
| Shown in menu/strip | Render with a "(missing)" suffix / placeholder. |
| Navigate to an orphan | Leave the current image unchanged; show a transient status message `Image not found: <fileName>` (reuse qView's existing error/status surface). |
| Save / delete / clear | Work normally regardless of orphans. |
| File reappears | Reactivates automatically on next access. |

## 9. UX, i18n, and Settings

- **Discoverability:** actions live in the View menu and Bookmarks submenu; nothing is
  keyboard-only.
- **Empty state:** Bookmarks submenu shows a disabled "No bookmarks" item (parity with an empty
  recents menu).
- **i18n:** every user-facing string goes through `tr()` and lands in `i18n/qview_*.ts`; no raw
  literals. (qView uses Weblate — translators pick up new strings automatically.)
- **Settings:** one new boolean, `SaveBookmarks` (default on). No other settings-dialog changes
  in v1.

## 10. Phasing

**v1 (this PRD's commitment):** data model, persistence, capture/restore, menu + submenu +
shortcuts, orphan handling, settings toggle, unit tests. No sidebar, no thumbnails, no reorder.

**v2 (follow-up):** bookmark **strip** in the merged sidebar (PR #838) showing cropped viewport
thumbnails (the `viewportSize` + `sceneCenter` + `zoom` already captured make the crop rect
derivable), delete-on-hover, drag-and-drop reorder, and a Clear-all footer — full SleekView UI
parity.

## 11. Resolved Decisions

The first draft left four questions open. All are resolved below against the current source; each
records the rationale so a maintainer can overrule with full context.

**Q1 — Scope: global flat list, not folder-scoped.** qView's only persistence precedent is the
recents list, and it is a single flat global list — `settings.value("recentFiles")` under group
`recents` ([actionmanager.cpp:330-355](../src/actionmanager.cpp#L330)). There is no folder-hashed
storage anywhere in the codebase; SleekView only hashed folder paths because it was sidecar-based,
and §5 already discards the sidecar. Folder keying would also accumulate dead keys for folders the
user never revisits, with no natural eviction point. Because every bookmark stores
`absoluteFilePath` regardless, a v2 "current folder only" view is a pure UI filter over the same
store — so this choice is reversible without a data migration.

**Q2 — Ship all three defaults; one macOS check outstanding.** None of `Ctrl+D`, `Ctrl+Up`, or
`Ctrl+Down` is bound anywhere in `shortcutmanager.cpp`. Note that **bare** `Up`/`Down` are Rotate
Right/Left ([shortcutmanager.cpp:129-136](../src/shortcutmanager.cpp#L129)); the modified pair
sitting next to rotation is deliberate and mnemonic, but reviewers should weigh the mis-keying
risk. `Ctrl+D` is the universal bookmark idiom and renders as `Cmd+D` on macOS under Qt's
modifier mapping.

**macOS `D`-key check — verified, no collision.** Probed against Qt 6.11.1 with a live
`QGuiApplication` on the cocoa platform theme: `QKeySequence::Delete` resolves to
`Del` (⌦) **and** `Meta+D` (native **⌃D**, i.e. physical Control+D). So qView's "Move to Trash"
([shortcutmanager.cpp:81-82](../src/shortcutmanager.cpp#L81)) does own a `D` combo on macOS.
But Qt maps `Qt::CTRL` to Command and `Qt::META` to physical Control on macOS, so
`saveBookmark` as `Qt::CTRL | Qt::Key_D` is **⌘D** — a distinct sequence. The two never collide.

Reviewers should still note the *adjacency*: ⌘D bookmarks, ⌃D moves the file to the trash — same
letter, one modifier apart, one of them destructive. Judged acceptable (⌘D is the idiom users
expect, and ⌃D is Qt's platform standard, not qView's choice), but if the maintainer disagrees the
fallbacks are `Ctrl+Shift+D` or shipping `saveBookmark` unbound.

**Q3 — Allow restore across files.** `QVGraphicsView::loadFile(QString)` is already public, and
the §8 orphan policy covers the missing-file case. Confining v1 to the current image would make
the bookmark list nearly redundant with ordinary zoom history; comparing regions across images is
the feature's main draw, and it matches SleekView.

**Q4 — Ignore frame position, and do not disturb playback.** `QMovie::jumpToFrame` is only
reliably forward-seeking for streamed formats, so exact frame restore would require decoding from
the start. Revised from the draft's "restore lands on frame 0": v1 should not touch the movie at
all. A freshly loaded file begins at frame 0 anyway, and an already-loaded animation should keep
playing where it is rather than being jolted backwards. Only zoom/pan/orientation are applied.

## 12. Testing

- **Unit tests** ([tests/](../tests/), `BUILD_TESTS`) for the pure logic — the only layer with
  existing coverage today ([tst_actionmanagertests.cpp](../tests/tst_actionmanagertests.cpp)):
  - serialize → deserialize round-trip (`QVariantList` ⇄ `QList<Bookmark>`), including the
    pre-existing-data / missing-field decode path, and including `mirrored`/`flipped` (a decode
    defaulting these to `false` on legacy data is correct and should be asserted).
  - next/previous index math with wraparound and the empty-list / single-item edge cases
    (mirrors SleekView's `nextFlatIndex` / `previousFlatIndex`).
  - orphan detection given a folder listing.
- **Manual verification:** save at several zoom/pan/rotation states, restart qView, confirm
  exact restoration; resize the window between save and restore and confirm the center holds.
  Explicitly cover the orientation matrix — bookmark while mirrored, while flipped, and while
  both mirrored and rotated 90°, then restore from a clean (un-mirrored) view and from an
  oppositely-mirrored view, to prove the toggle-vs-set logic in §6.2 step 3.
- **Formatting/CI:** `./build.sh --format-check` must pass; feature builds clean on Qt 5 and
  Qt 6 (the codebase supports both).

## 13. Files Likely Touched

| File | Change |
|---|---|
| `src/bookmarkmanager.{h,cpp}` *(new)* | Bookmark struct, list, persistence, nav index, orphan logic |
| [src/qvgraphicsview.h](../src/qvgraphicsview.h) / `.cpp` | `getViewportState` / `applyViewportState` |
| [src/actionmanager.cpp](../src/actionmanager.cpp) | Register actions; build Bookmarks submenu |
| [src/settingsmanager.h](../src/settingsmanager.h) / `.cpp` | `SaveBookmarks` setting |
| [src/shortcutmanager.cpp](../src/shortcutmanager.cpp) | Default shortcuts |
| [src/mainwindow.cpp](../src/mainwindow.cpp) | Wire actions ↔ view ↔ manager |
| [CMakeLists.txt](../CMakeLists.txt) | Add new sources |
| `tests/` | Unit tests for the new logic |
| `i18n/` | Strings picked up automatically |

## 14. Appendix — Unrelated Defect Found During the Audit

Not part of this feature; recorded because it lives in a file this feature touches and should be
fixed in its own PR.

Commit `d0c945a` ("support titlebar hiding on macos in a different way…") removed the
`fullscreen` and `toggletitlebar` entries from `ShortcutManager::initializeShortcutsList`, but
left the platform-specific fixups that followed them
([shortcutmanager.cpp:142-152](../src/shortcutmanager.cpp#L142)). Those fixups mutate
`shortcutsList.last()`, which is now **`flip`**. Present in `jurplel/qView@main` as of `c5eca1c`.

Consequences:
- `fullscreen` has no default shortcut **and** is absent from `shortcutsList`, so it cannot be
  rebound in the shortcut editor at all — though the action itself still exists and is in the
  View menu ([actionmanager.cpp:748](../src/actionmanager.cpp#L748)). The binding it lost is
  `QKeySequence::FullScreen`, which on Qt 6.11.1/macOS is `Meta+Ctrl+F` (native **⌃⌘F**);
  on Windows it is `F11`.
- On Windows, `Alt+Return` is now registered as a default binding for **Flip**.
- On Linux, the `F11`/GNOME block tests `flip`'s bindings for `Ctrl+F11`, which is never true —
  dead code rather than a misbinding.
