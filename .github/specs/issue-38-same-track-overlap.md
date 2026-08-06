# Spec: Prevent Same-Track Timeline Item Overlap

- Status: Ready for agent
- Issue: [#38 — 同一轨道的两个素材可以重叠](https://github.com/0xsline/OpenChatCut/issues/38)
- Area: Timeline editing
- Decision date: 2026-08-06

## Problem Statement

An editor can currently place two timeline items over the same frame range on one video or audio track. This is especially visible when media is dragged directly from Windows Explorer, when insert placement lands inside an existing item, or when an existing item is moved or extended. The timeline keeps both items, paints an overlap warning, and relies on incidental item ordering to decide what is seen or heard.

From the user's perspective, this makes insert and overwrite placement appear equivalent, makes the resulting edit difficult to predict, and differs from the one-item-per-track-lane behavior expected from editors such as CapCut. It can also make transitions ambiguous because a transition expects a binary seam between adjacent items.

The problem is broader than one drag-and-drop adapter. Placement policy is optional at several call sites, insert placement does not split an item that spans the insertion frame, move and retime edits can bypass collision handling, and final media metadata can enlarge a provisional item after it has been placed. Fixing only the Windows drop handler would leave the same invalid state reachable through other editing paths.

## Solution

Make non-overlapping occupancy a forward invariant for every video and audio track. Timeline item ranges use half-open intervals: two items may touch at an edge, but their occupied frames may not intersect.

All geometry-changing edits will pass through one lane-edit planning module before they are committed. Its interface will accept a discriminated edit request and return either one atomic next state or a structured rejection. The interface will require an explicit collision policy whenever a frame position is supplied:

- **Insert** preserves all existing content. If the insertion frame is inside an item, that item is split at the frame and its right fragment, plus later affected items, moves right by the inserted duration.
- **Overwrite** preserves timeline positions outside the placed range. Existing items intersecting the range are removed, trimmed, or split so the new item exclusively occupies that range; later items do not move.
- **Constrain** is non-destructive and is used for moving and trimming existing items. The requested edit resolves to a legal gap or edge; if the selection cannot fit, the edit is rejected and the timeline remains unchanged.
- **Append** places an item at the current end of a compatible track and is the explicit policy for callers that do not target a frame.

For example, if item A occupies `[0, 100)` and a 30-frame item B is placed at frame 40:

| Policy | Result |
| --- | --- |
| Insert | A-left `[0, 40)`, B `[40, 70)`, A-right `[70, 130)`; later affected items also move 30 frames. |
| Overwrite | A-left `[0, 40)`, B `[40, 70)`, A-right `[70, 100)`; later item positions do not move. |
| Constrain | Placement succeeds only at a legal gap or edge large enough for B; otherwise the edit is rejected. |

Visual and audio layering remains available by using different compatible tracks. A normal drop will not silently create a new track or preserve an overlap on the current track.

Existing projects that already contain overlaps must continue to open without destructive normalization. The editor will prevent new or enlarged overlaps while allowing edits that reduce or remove legacy overlaps. Automatic migration of legacy overlaps is not part of the first implementation.

## User Stories

1. As a Windows desktop editor, I want media dragged from Explorer to honor the selected placement mode, so that external files behave like media-pool items.
2. As a macOS desktop editor, I want media dragged from Finder to use the same placement semantics, so that behavior is platform-independent.
3. As a media-pool user, I want every timeline drop to require an explicit placement mode, so that an omitted option cannot accidentally create an overlap.
4. As an editor using insert placement, I want a clip under the insertion frame to be split, so that no source content is lost.
5. As an editor using insert placement, I want the right fragment and later affected clips to move by the inserted duration, so that the inserted clip has exclusive space.
6. As an editor inserting at an existing seam, I want the later clip to move without creating a zero-length fragment, so that the edit remains clean.
7. As an editor inserting into an empty gap, I want only clips at or after the insertion frame to move, so that earlier timing is preserved.
8. As an editor using overwrite placement, I want a fully covered clip to be removed, so that only the new clip occupies the overwritten frames.
9. As an editor using overwrite placement, I want a partially covered clip to be trimmed at the correct edge, so that unaffected content remains in place.
10. As an editor overwriting the middle of a clip, I want the clip split around the overwritten range, so that content before and after the range remains available.
11. As an editor using overwrite placement, I want later clips to retain their frame positions, so that overwrite never behaves like insert.
12. As an editor who needs picture-in-picture or simultaneous graphics, I want to place the items on separate compatible tracks, so that layer order is explicit.
13. As an editor moving one clip, I want the move to resolve to a legal gap or boundary, so that moving cannot create a same-track overlap.
14. As an editor moving a clip across tracks, I want the target track checked while the source item is excluded, so that valid cross-track moves succeed.
15. As an editor moving multiple selected clips, I want their shared offsets preserved, so that collision handling does not distort the selection.
16. As an editor moving a linked or sync-locked selection, I want the edit to be atomic, so that one member cannot move when another member is blocked.
17. As an editor trimming a clip edge, I want a normal trim constrained by the neighboring item, so that extending a clip cannot cover its neighbor.
18. As an editor using ripple trim, I want later affected clips shifted consistently, so that the track remains collision-free while the seam follows the trim.
19. As an editor using rate stretch, I want the stretched range constrained or rippled according to the edit mode, so that speed changes cannot introduce overlaps.
20. As an editor using snapping, I want snapped positions evaluated against legal occupancy, so that a snap target never commits an invalid placement.
21. As an editor working on a locked track, I want any placement that would change the locked track rejected atomically, so that locks remain trustworthy.
22. As an editor working with transitions, I want only adjacent, non-overlapping base items to define a seam, so that transition endpoints stay unambiguous.
23. As an editor working with source-trimmed media, I want splits and trims to preserve the correct source window, so that the retained frames do not change unexpectedly.
24. As an editor working with transcript-driven audio, I want lane edits to preserve edited-word timing semantics, so that audio content remains synchronized.
25. As an editor working with keyframed clips, I want inserted and overwritten fragments to retain correctly partitioned keyframes, so that animation remains visually continuous.
26. As an editor working with fades, I want fragment durations to reconcile fade handles, so that a lane edit does not create invalid fade geometry.
27. As an editor importing several files, I want items of each compatible kind packed deterministically, so that batch order remains stable and collision-free.
28. As an editor waiting for media probing, I want final duration metadata reconciled through the original placement policy, so that a provisional clip cannot later grow over another clip.
29. As an editor who moves a provisional import manually, I want later metadata updates to avoid moving unrelated clips unexpectedly, so that my manual edit remains authoritative.
30. As an editor whose import fails or is cancelled, I want provisional items removed without disturbing unrelated geometry, so that cleanup cannot create gaps or secondary moves beyond the failed batch.
31. As an editor using Add to Timeline, paste, templates, sound effects, or voice-over insertion, I want those entry points to use the same occupancy rules, so that the result does not depend on how the item was created.
32. As an Agent user, I want timeline tools to return a structured collision error or a planned legal result, so that automated edits cannot silently corrupt lane geometry.
33. As an Agent user reviewing a proposal, I want the preview and committed result produced by the same planner, so that approval cannot change placement semantics.
34. As an editor, I want a rejected drag or trim to leave the project unchanged, so that invalid edits do not partially mutate clips, transitions, links, or selection.
35. As an editor, I want one accepted lane edit to occupy one undo step, so that split, shift, trim, and placement can be reversed together.
36. As an editor, I want redo to recreate the exact collision-free result, so that history is deterministic.
37. As an editor, I want the active insert or overwrite mode to remain visibly selected and accessible by keyboard and assistive technology, so that I can predict the next drop.
38. As an editor, I want collision feedback during a drag, so that I can see whether the pending edit will insert, overwrite, constrain, or be rejected before releasing it.
39. As an editor opening an older project with overlaps, I want the project to render as before, so that adopting the new rule does not destroy previous work.
40. As an editor repairing a legacy overlap manually, I want legal moves and deletions to remain available, so that I can reduce invalid geometry incrementally.
41. As an editor previewing or exporting a project, I want preview and export to use the same resolved item geometry, so that eliminating overlaps does not create output differences.
42. As an editor of a long timeline, I want collision checks limited to affected tracks and ranges, so that the new rule does not make dragging noticeably slower.

## Implementation Decisions

- Introduce one deep, in-process lane-edit planning module. Its external seam is a single editor-command interface used by UI adapters, Agent tools, proposal drafts, and tests. Complex interval, split, ripple, overwrite, link, transition, and history behavior remains inside the module.
- Use a discriminated request rather than optional `ripple` and `overwrite` booleans. A caller that supplies a target frame must also supply `insert`, `overwrite`, or `constrain`; a caller without a target frame must explicitly request `append`.
- Return a structured result containing either the atomic next timeline state and edit summary or a rejection reason such as locked track, incompatible track, collision, insufficient gap, invalid source range, or stale import state. Callers must not infer success from a silent no-op.
- Treat item occupancy as a half-open range. Edge contact is legal; any positive-length intersection between two items on the same video or audio track is illegal for newly committed geometry.
- Ignore transition render handles when calculating occupancy. Transitions remain metadata on a binary seam between adjacent base ranges and may render source frames outside those base ranges without making the items overlap.
- Reuse the existing overwrite behavior as the overwrite implementation: fully covered items are removed, edge intersections are trimmed, middle intersections are split around a hole, and the new item is added last in one atomic edit.
- Implement insert as source-aware split plus ripple. When an item spans the insertion frame, split it first, shift the right fragment and later affected target-lane items by the inserted duration, and then place the new item in the reserved range.
- Preserve current synchronization semantics. Ripple shifts propagate through existing sync-lock relationships. Lane-local destructive operations remove or repair stale link membership rather than implicitly cutting unrelated companion tracks.
- Implement constrain against the occupancy of non-moving items. A move or trim may land in any legal gap that can contain the edited range; otherwise it resolves to the nearest legal boundary or rejects without changing state. Multi-item edits preserve one shared frame delta and track offset.
- Keep low-level add, move, retime, and full-state replacement operations as implementation details for trusted planners and migrations. User-facing and Agent-facing adapters must not use them to bypass lane planning.
- Add a commit guard that detects new or enlarged same-track intersections. The guard allows a legacy intersection to remain unchanged or shrink, which preserves old projects while preventing further invalid geometry.
- Keep the existing overlap-span calculation for legacy-state warnings and pending-drag feedback. A successfully committed edit from a valid state must not need a permanent overlap warning.
- Store the active placement mode at an editor-level seam shared by all placement adapters. Media-pool drops, library items, external files, add-at-playhead actions, paste, voice-over placement, and other frame-targeted actions must receive the same explicit mode.
- Preserve progressive import. A provisional item records its placement policy and anchor. When authoritative duration arrives, duration reconciliation goes through the lane-edit module against live state instead of directly extending the item.
- If a user manually moves or retimes a provisional item, automatic batch reflow no longer owns its geometry. Final metadata reconciliation uses the non-destructive constrain policy and may shorten the placed source window with user-visible feedback rather than moving unrelated items.
- Keep media ingest lifecycle changes separate from lane-edit history, but commit every synchronous split, trim, shift, remove, and place result for one lane edit as a single undoable state change.
- Preserve selection on the newly placed or moved items after a successful edit. A rejected edit preserves the previous selection and project state.
- Reconcile transitions, source windows, transcripts, keyframes, fades, link groups, and selected IDs inside the planner before returning success.
- Limit occupancy work to affected video or audio tracks. The implementation may build sorted track-local intervals per request and inspect only neighbors and intersections relevant to the requested ranges.
- Do not automatically create a new track when an overlap would occur. Layer creation remains an explicit user action in the first implementation.
- Do not change caption-lane behavior. Caption cues already use lane-specific neighbor constraints and remain governed by their existing module.

## Testing Decisions

- The highest behavioral test seam is the editor-command interface backed by the existing in-memory draft engine. Tests issue a lane-edit request and assert on the observable timeline, history, selection, and structured result rather than calling reducer internals directly.
- The pure planning implementation may have an internal seam for an exhaustive geometry matrix, but those tests supplement rather than replace tests through the editor-command interface.
- Good tests assert frame ranges, source windows, preserved content, transitions, links, selection, history, and rejection behavior. They must not assert the private sequence of reducer actions when multiple valid implementations produce the same result.
- Extend the existing overwrite verification as prior art for the four intersection regions, source-aware fragments, linked-item cleanup, transition endpoint reconciliation, locked-track rejection, and one-step undo.
- Use the existing caption movement constraints as prior art for move and trim clamping, while testing video and audio occupancy through the new lane-edit seam.
- Use the existing external-file-drop and media-placement verification as prior art for platform-independent drop classification, batch order, provisional items, and final-duration reflow.
- Add a placement-mode matrix covering media-pool drag, external-file drag, template and sound placement, add at playhead, paste, voice-over, and Agent placement. Each adapter test needs only to prove that the correct explicit request reaches the common seam.
- Cover insert at an empty frame, exact seam, item start, item end, and item middle. Assert no zero-length fragments and no lost source content.
- Cover overwrite with no intersection, full containment, left-edge intersection, right-edge intersection, and middle-hole intersection. Assert later item positions remain unchanged.
- Cover single-item, linked-item, sync-locked, multi-selection, and cross-track moves. Assert one shared delta, atomic rejection, and no new overlap on any affected track.
- Cover normal trim, ripple trim, and rate stretch at both edges. Assert the correct neighbor constraint or ripple shift and valid source bounds.
- Cover touching intervals explicitly to prevent an off-by-one rule that treats a shared edge as an overlap.
- Cover transitions before, at, and after edited ranges. Assert that valid binary seams survive and invalid or dangling transitions are removed deterministically.
- Cover provisional imports whose final duration grows and shrinks, including a user move during import, cancellation, failure, and multiple files completing out of order.
- Cover legacy states with an existing overlap. Assert that loading is unchanged, unrelated metadata edits remain possible, new or enlarged intersections are rejected, and edits that reduce the overlap succeed.
- Add a deterministic invariant test that applies representative sequences of place, move, trim, rate-stretch, duration-reconcile, undo, and redo operations from a valid state and checks every affected video and audio track for positive-length intersections after each accepted edit.
- Run the focused lane-edit, overwrite, pointer, external-drop, media-placement, transition, professional-timeline, and history verification suites. The repository typecheck and lint must also pass before implementation is considered complete.

## Out of Scope

- Automatically migrating existing overlapping items onto newly created tracks.
- Automatically creating a new track when a drop would overlap the current track.
- Adding a free-overlap or compositing mode to a single track.
- Redesigning track ordering, track headers, or the general timeline toolbar beyond making placement mode and rejection feedback unambiguous.
- Changing cross-track visual compositing or audio mixing semantics.
- Changing transition rendering, transition duration rules, or source-handle generation beyond reconciling endpoints after a lane edit.
- Changing caption cue collision behavior.
- Combining the complete asynchronous media ingest lifecycle into one undo record.
- Rewriting old project data merely because it contains legacy overlaps.

## Further Notes

- The current overwrite planner already demonstrates that collision-free placement can preserve source windows, transitions, link validity, and single-step undo. The implementation should deepen that behavior behind the common lane-edit interface rather than adding more adapter-specific checks.
- The current overlap hatch is useful evidence for legacy data and for a pending invalid drag, but it should not be the steady-state resolution for a newly accepted edit.
- The issue report does not identify an exact released version. The failure is nevertheless reproducible in the current code paths and is not limited to a historical package.
- This spec deliberately favors predictable, non-destructive movement. A later product decision may add modifier-key overwrite moves or automatic overlay-track creation, but those behaviors should be new explicit policies rather than exceptions to the occupancy invariant.
