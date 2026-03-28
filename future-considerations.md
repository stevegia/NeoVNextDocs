# Future Considerations

A scratchpad for deferred ideas that are worth building eventually but not now.

---

## ComfyUI Job Checkpoint Tracking

**Context:** The "Generation Progress" section was removed from the Job detail panel (2026-03-26)
because no reliable per-step progress data is available yet — ComfyUI's progress events arrive
over an SSE stream but the stream is not persisted, making it useless for historical job views
and noisy for the current panel design.

**What would be useful:**
- Track ComfyUI node execution checkpoints as persisted events in the database (e.g. a
  `job_checkpoints` table keyed by job ID + node ID + timestamp).
- The worker already emits `progress` and `executing` events via the SSE stream
  (`streamJobEvents`). The missing piece is writing those to storage.
- The frontend hook `useComfyProgress` already consumes the live stream. It could be extended
  to hydrate from persisted checkpoints when the job is completed/historical.

**Acceptance criteria:**
- Completed comfy jobs show a replay of node execution steps in the detail panel.
- Running jobs show a live updating list of completed nodes with step count (e.g. "KSampler: 14/20").
- Timed-out or failed jobs show the last checkpoint before failure to aid debugging.

**Related files:**
- `src/frontend/src/hooks/useComfyProgress.ts` — live SSE consumer
- `src/frontend/src/components/ComfyJobProgress.tsx` — rendering (currently not displayed in the panel)
- `src/frontend/src/pages/JobsPage.tsx` — detail panel host

---
