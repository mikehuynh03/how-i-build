# Phase 3: Gesture Interactions — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development + superpowers:test-driven-development for the detectors. Before any Next/React edits, consult `node_modules/next/dist/docs/01-app/`.

**Goal:** Add the new gesture detectors (two-finger, fist, thumbs-up) and emit the interaction events that drive the canvas — grab/move, pan, connect, delete-dwell, ink, two-hand zoom, radial-menu open — wiring them to the existing store op set.

**Architecture:** New pure detectors live in `lib/gesture-engine/gestures.ts` and are validated against constructed fixture landmarks (TDD, no camera). `useGestureEvents` gains new event variants following the existing per-frame precedence discipline. A new `lib/canvas/gesture-bridge.ts` (headless, testable) maps a `GestureEvent` + current store state → store op calls, so the mapping is unit-testable without a camera. The radial menu is a Framer-Motion component.

**Gesture contract:** every new gesture is added to `docs/gesture-contract.md` FIRST (it is the single source of truth), then implemented, then tested — same discipline as the ported engine.

**Live-tuning gate:** thresholds (curl margins, dwell frames, connect hysteresis) are seeded from reasonable defaults here but MUST be confirmed/tuned on a real webcam with the user before Phase 3 is marked done. Unit tests lock the *logic*; live testing tunes the *constants*.

**Work from:** `~/Desktop/claude/renewed-gesture-testing`.

---

### Task 1: New gesture detectors (TDD, camera-independent)

**Files:** Modify `lib/gesture-engine/gestures.ts`, `lib/gesture-engine/constants.ts`, `docs/gesture-contract.md`; Create `tests/gestures-phase3.spec.ts`.

Landmark indices (MediaPipe): wrist 0; thumb 1-4 (TIP 4); index 5-8 (PIP 6, TIP 8); middle 9-12 (PIP 10, TIP 12); ring 13-16 (PIP 14, TIP 16); pinky 17-20 (PIP 18, TIP 20). y is normalized, 0=top, 1=bottom. A finger is **extended** when its TIP is above (smaller y than) its PIP; **curled** when TIP is below (larger y than) its PIP — same convention as the existing `detectPalm`.

- [ ] **Step 1: Add the contract section** to `docs/gesture-contract.md` (append under a new "## Phase 3 gestures" heading): define `detectTwoFinger` (index+middle extended, ring+pinky curled → used for "connect"; cursor is the midpoint of landmarks 8 and 12), `detectFist` (all four fingers curled → "delete", dwell-confirmed), `detectThumbsUp` (four fingers curled AND thumb tip 4 above the knuckle row, i.e. `y(4) < y(5)` by `THUMBSUP_MARGIN` → opens the radial menu). Add constant rows for `CURL_MARGIN`, `THUMBSUP_MARGIN`, `DELETE_DWELL_FRAMES`, `CONNECT_DEBOUNCE_FRAMES`.

- [ ] **Step 2: Add constants** to `lib/gesture-engine/constants.ts`:
```ts
export const CURL_MARGIN = 0.02;            // tip must be at least this far BELOW pip (normalized) to count as curled
export const THUMBSUP_MARGIN = 0.04;        // thumb tip must be this far above the index MCP to count as "up"
export const DELETE_DWELL_FRAMES = 15;      // fist held steady ~0.5s @30fps to commit a delete
export const CONNECT_DEBOUNCE_FRAMES = 3;   // two-finger must hold this many frames before connect-start
```

- [ ] **Step 3: Write failing tests** in `tests/gestures-phase3.spec.ts`. Build landmark arrays from a helper so intent is explicit:
```ts
import { describe, it, expect } from "vitest";
import { detectTwoFinger, detectFist, detectThumbsUp } from "@/lib/gesture-engine/gestures";

type LM = { x: number; y: number; z: number };
// 21 landmarks; default everything mid-frame, then override per finger.
function hand(overrides: Partial<Record<number, Partial<LM>>> = {}): LM[] {
  const base: LM[] = Array.from({ length: 21 }, () => ({ x: 0.5, y: 0.5, z: 0 }));
  for (const [i, v] of Object.entries(overrides)) base[+i] = { ...base[+i], ...v } as LM;
  return base;
}
// helper: set a finger extended (tip above pip) or curled (tip below pip)
function finger(o: Record<number, Partial<LM>>, pip: number, tip: number, mode: "ext" | "curl") {
  o[pip] = { y: 0.5 };
  o[tip] = { y: mode === "ext" ? 0.40 : 0.60 }; // ext: tip above; curl: tip below pip by >CURL_MARGIN
}

describe("detectTwoFinger", () => {
  it("true when index+middle extended and ring+pinky curled", () => {
    const o: Record<number, Partial<LM>> = {};
    finger(o, 6, 8, "ext"); finger(o, 10, 12, "ext"); finger(o, 14, 16, "curl"); finger(o, 18, 20, "curl");
    expect(detectTwoFinger(hand(o))).toBe(true);
  });
  it("false when all fingers extended (open hand)", () => {
    const o: Record<number, Partial<LM>> = {};
    finger(o, 6, 8, "ext"); finger(o, 10, 12, "ext"); finger(o, 14, 16, "ext"); finger(o, 18, 20, "ext");
    expect(detectTwoFinger(hand(o))).toBe(false);
  });
  it("false when only index extended (point)", () => {
    const o: Record<number, Partial<LM>> = {};
    finger(o, 6, 8, "ext"); finger(o, 10, 12, "curl"); finger(o, 14, 16, "curl"); finger(o, 18, 20, "curl");
    expect(detectTwoFinger(hand(o))).toBe(false);
  });
});

describe("detectFist", () => {
  it("true when all four fingers curled", () => {
    const o: Record<number, Partial<LM>> = {};
    finger(o, 6, 8, "curl"); finger(o, 10, 12, "curl"); finger(o, 14, 16, "curl"); finger(o, 18, 20, "curl");
    expect(detectFist(hand(o))).toBe(true);
  });
  it("false when any finger extended", () => {
    const o: Record<number, Partial<LM>> = {};
    finger(o, 6, 8, "curl"); finger(o, 10, 12, "ext"); finger(o, 14, 16, "curl"); finger(o, 18, 20, "curl");
    expect(detectFist(hand(o))).toBe(false);
  });
});

describe("detectThumbsUp", () => {
  it("true when fingers curled and thumb tip above the index MCP", () => {
    const o: Record<number, Partial<LM>> = {};
    finger(o, 6, 8, "curl"); finger(o, 10, 12, "curl"); finger(o, 14, 16, "curl"); finger(o, 18, 20, "curl");
    o[5] = { y: 0.5 }; o[4] = { y: 0.42 }; // thumb tip well above index MCP (0.5 - 0.42 = 0.08 > THUMBSUP_MARGIN)
    expect(detectThumbsUp(hand(o))).toBe(true);
  });
  it("false when fingers curled but thumb not raised (plain fist)", () => {
    const o: Record<number, Partial<LM>> = {};
    finger(o, 6, 8, "curl"); finger(o, 10, 12, "curl"); finger(o, 14, 16, "curl"); finger(o, 18, 20, "curl");
    o[5] = { y: 0.5 }; o[4] = { y: 0.5 };
    expect(detectThumbsUp(hand(o))).toBe(false);
  });
});
```

- [ ] **Step 4: Run, verify fail** — `npm test -- tests/gestures-phase3.spec.ts`.

- [ ] **Step 5: Implement detectors** in `lib/gesture-engine/gestures.ts` (append; import the new constants):
```ts
import { CURL_MARGIN, THUMBSUP_MARGIN } from "./constants";
// reuse the existing HandLandmark type already in this file

function isExtended(lm: { y: number }[], pip: number, tip: number): boolean {
  return lm[tip].y < lm[pip].y; // tip above pip
}
function isCurled(lm: { y: number }[], pip: number, tip: number): boolean {
  return lm[tip].y > lm[pip].y + CURL_MARGIN; // tip clearly below pip
}

export function detectTwoFinger(lm: { y: number }[]): boolean {
  return isExtended(lm, 6, 8) && isExtended(lm, 10, 12) && isCurled(lm, 14, 16) && isCurled(lm, 18, 20);
}
export function detectFist(lm: { y: number }[]): boolean {
  return isCurled(lm, 6, 8) && isCurled(lm, 10, 12) && isCurled(lm, 14, 16) && isCurled(lm, 18, 20);
}
export function detectThumbsUp(lm: { y: number }[]): boolean {
  const fingersCurled = isCurled(lm, 6, 8) && isCurled(lm, 10, 12) && isCurled(lm, 14, 16) && isCurled(lm, 18, 20);
  const thumbUp = lm[4].y < lm[5].y - THUMBSUP_MARGIN;
  return fingersCurled && thumbUp;
}
```
(Match the file's existing `HandLandmark` type rather than the inline `{y:number}[]` if it differs — adapt signatures to the real type used by the other detectors.)

- [ ] **Step 6: Run, verify pass.** Then run the FULL suite `npm test` (expect 61 + new ones green) and `npx tsc --noEmit`.
- [ ] **Step 7: Commit** — `git commit -m "feat: two-finger, fist, thumbs-up gesture detectors + contract"`.

---

### Task 2: Emit interaction events from useGestureEvents (camera-independent logic)

**Files:** Modify `lib/gesture-engine/types.ts` (extend `GestureEvent` union), `lib/gesture-engine/useGestureEvents.ts`. Test: extend `tests/` with a state-machine test if the emission logic is extracted to a pure helper.

Add event variants: `{type:"connect-start"|"connect-move"|"connect-end"; x; y}` (cursor at midpoint of landmarks 8 & 12, debounced by `CONNECT_DEBOUNCE_FRAMES`), `{type:"delete-dwell"; progress}` + `{type:"delete-commit"}` (fist held, reusing the existing dwell-tracker with `DELETE_DWELL_FRAMES`), `{type:"radial-open"; x; y}` (thumbs-up, debounced, emitted once per gesture), `{type:"zoom"; scale; cx; cy}` (both hands present: ratio of current inter-hand distance to the distance at zoom-start, about the midpoint).

Per-frame precedence (extend the existing one): **palm → idle** (wins) → **two hands present → zoom** → **thumbs-up → radial-open** → **fist → delete-dwell** → **two-finger → connect** → **pinch → draw/grab (existing)** → **cursor**. Keep the existing single-hand pinch/palm/cursor paths intact. Extract the precedence resolution into a pure function if feasible so it can be unit-tested with sequences of fixture frames.

- [ ] Steps: add union variants → write a pure `resolveGesture(frame, prevState)` (or extend existing) with unit tests for precedence ordering → wire into the hook → `npm test` + `tsc` green → commit. (Full step-by-step to be expanded by the implementer following the existing useGestureEvents structure; the contract section from Task 1 is authoritative for trigger conditions.)

---

### Task 3: Gesture→op bridge (headless, testable)  ·  Task 4: Radial menu  ·  Task 5: Live tuning

These wire events to the store and add the radial menu. **Task 3** (`lib/canvas/gesture-bridge.ts`): a pure reducer `applyGesture(store, event, hover)` mapping events → ops (cursor→hover hit-test; pinch on element→move, on empty→pan; connect-end on a node→`link`; delete-commit→`remove(hover)`; ink tool + draw→accumulate points then `addInk`; zoom→`zoomAt`; radial-open→open menu state). Unit-test the bridge with a fake store. **Task 4** (`components/canvas/RadialMenu.tsx`): Framer-Motion radial of 6 shapes + note + text; index cursor selects a wedge, pinch commits, spawns at cursor. **Task 5**: live webcam checklist in `docs/test-checklist.md` covering every gesture, tuning the Task-1/Task-2 constants with the user. **Task 5 is the gate that marks Phase 3 done.**

---

## Self-Review

**Spec coverage:** detectors (T1) + event emission (T2) + bridge/radial (T3/T4) cover the proposed gesture vocabulary (grab/move, pan, connect, delete-dwell, ink, zoom, radial creation). Live validation (T5) is the explicit done-gate, honoring verification-before-completion for camera-dependent behavior.

**Placeholder note:** Task 1 is fully specified (TDD code complete). Tasks 2-5 are specified at the interface/contract level with the gesture-contract as authoritative source; they are intentionally expanded by the implementer against the live engine structure AND require user gesture-vocab sign-off + webcam tuning before execution — this is a deliberate checkpoint, not an omission. Task 1 is the only task dispatched until that checkpoint clears.

**Consistency:** constants (`CURL_MARGIN`, `THUMBSUP_MARGIN`, `DELETE_DWELL_FRAMES`, `CONNECT_DEBOUNCE_FRAMES`) defined T1, referenced T2; store ops referenced in T3 match Phase 2's store API exactly.
