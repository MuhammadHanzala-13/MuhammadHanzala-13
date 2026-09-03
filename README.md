- 👋 Hi, I’m Muhammad Hanzala 
- 👀 I’m interested in exploring new tech
- 🌱 I’m currently learning ML 
- 💞️ I’m looking to collaborate on
- 📫 How to reach me -> https://github.com/MuhammadHanzala-13
- 📫 How to reach me -> https://www.linkedin.com/in/muhammadhanzala13
<!---
muhammadhanzala-13/muhammadhanzala-13 is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
 # GymZAI Pose Estimation Ruthless Review

**Review date:** 2026-09-03  
**Scope:** Read-only audit of the pose detection pipeline, exercise selection, posture tracking, rep counting, UI integration, and documented contracts.  
**Code changes:** None in this review.  
**Dependency changes:** None recommended yet.

## Executive Finding

The feature is not currently bulletproof. The most likely root cause of completely incorrect or missing detection is the camera callback contract in `services/camera/frameProcessor.ts`: the installed MediaPipe package documents pose data as `result.landmarks`, while the application reads `poseResults.results[0]?.landmarks?.[0]`. If the runtime payload follows the installed package contract, the engine receives no usable landmark frame.

Even after that integration issue is corrected, the engine is not ready for reliable rep counting or posture tracking. It lacks temporal filtering, stable body-side selection, complete form checks, push-up support, deterministic exercise/variation validation, and focused automated tests.

## Severity Legend

- **Critical:** Can prevent detection entirely or make the selected exercise fundamentally wrong.
- **High:** Can produce false counts, missed counts, or unsafe/incorrect form feedback during normal use.
- **Medium:** Produces inconsistent results, misleading UI, or incomplete session data.
- **Low:** Contract, maintainability, or release-readiness problem that still needs resolution.

## Critical Flaws

### C1. Camera result payload is likely read from the wrong property

**File:** `services/camera/frameProcessor.ts`  

The installed package documentation uses:

```ts
result.landmarks
```

The application defines and reads:

```ts
poseResults.results[0]?.landmarks?.[0]
```

This can cause `rawLandmarks` to be undefined on every callback, so `adaptLandmarks()` and `processFrame()` are never reached.

**Impact:** No reliable engine input, no rep updates, no posture analysis, and potentially no skeleton overlay.

**Confirm with:** Log only the top-level keys and landmark-array length of one callback result, without logging camera frames or full landmark streams.

### C2. Push-up is declared but not implemented

**Files:** `types/pose.ts`, `services/pose/poseEngine.ts`, `src/screens/PoseExerciseSelectScreen.js`

The shared types and project contract declare `pushup`, but `poseEngine.ts` only dispatches to bicep-curl logic or squat logic. Any push-up passed to the engine falls through to squat processing.

**Impact:** A push-up session cannot count push-up reps correctly and can report squat angles/form states for a different exercise.

**Required decision:** Align the product scope before implementation: `squat + push-up` as stated by the POC documents, or `squat + bicep curl` as currently exposed by the UI.

### C3. There is no reliable proof that the incoming coordinate convention matches the engine and overlay

**Files:** `services/camera/frameProcessor.ts`, `components/pose/SkeletonOverlay.tsx`, `services/pose/angleCalculator.ts`

The camera options force portrait output and front-camera mirroring. The overlay then applies a manual axis swap, inversion, and mirror correction. The engine calculates angles from the raw coordinates, not from the displayed coordinates.

If the native package already returns display-oriented or mirrored coordinates, the overlay can be visually displaced. If the raw coordinates are camera-oriented, the angle math may still be valid in principle, but this has not been verified with a known landmark fixture.

**Impact:** The skeleton can appear in the wrong place, and UI appearance cannot be trusted as evidence that the engine is measuring the intended anatomy.

**Confirm with:** Use a deterministic known-pose frame and verify shoulder, hip, knee, elbow, and wrist positions in both raw coordinates and rendered screen coordinates.

## High-Severity Flaws

### H1. Raw landmark stream is not smoothed or outlier-filtered

**Files:** `services/camera/frameProcessor.ts`, `services/pose/poseEngine.ts`

The pipeline attempts to process every 16 ms, approximately 60 Hz, and sends raw landmark coordinates directly into angle calculations. The engine has no moving average, One Euro filter, temporal median, velocity limit, or hysteresis.

**Impact:** MediaPipe jitter can repeatedly cross FSM thresholds, causing false transitions, missed transitions, unstable angles, and false form failures.

The engine contract specifies a throttled 5-10 Hz stream, but the implementation targets 60 FPS.

### H2. Squat body-side selection can change during a repetition

**File:** `services/pose/poseEngine.ts`

The engine chooses left or right squat landmarks independently on every frame based on average visibility. A small visibility fluctuation can switch sides in the middle of a rep.

**Impact:** The measured hip angle can jump between two different anatomical views, producing false depth transitions or corrupted rep timing.

A side should be selected during calibration or standing and locked until the rep/session is safely reset.

### H3. Squat posture checks promised by configuration are not implemented

**Files:** `constants/poseConfig.ts`, `services/pose/poseEngine.ts`

Configuration declares `kneeValgusDeviationDeg` and `spineAlignmentMinDeg`, but the squat engine does not calculate knee valgus or spine alignment. Heel-rise checking is also absent.

**Impact:** The system can count a geometrically deep squat while failing to detect important posture problems.

### H4. Curl elbow-flare geometry is not anatomically stable

**File:** `services/pose/poseEngine.ts`

The curl check compares normalized screen-space horizontal and vertical displacement:

```ts
upperArmDx > upperArmDy * threshold
```

This is dependent on camera orientation, body rotation, mirroring, distance, and perspective. It does not measure a stable anatomical elbow/upper-arm angle.

**Impact:** Reasonable curls can fail, while actual swinging or flare can be missed.

### H5. Visibility handling can accept unknown confidence as perfect confidence

**File:** `services/camera/landmarkAdapter.ts`

Missing `visibility` values default to `1.0`:

```ts
visibility: lm.visibility ?? 1.0
```

Unknown confidence should not be treated as maximum confidence.

**Impact:** Incomplete native data can pass confidence gates and produce invalid angle measurements.

### H6. Invalid exercise and variation combinations are silently accepted

**Files:** `services/pose/index.ts`, `services/pose/poseEngine.ts`, `types/pose.ts`

The public API accepts the union of all variations independently from the exercise. For example, a squat can be started with `hammer`, or a push-up with `sumo`. Runtime lookup then falls back to unrelated standard configuration.

**Impact:** The session can display one exercise while applying another exercise's thresholds.

The API needs a discriminated exercise/variation contract or explicit runtime validation that rejects invalid combinations.

### H7. Pause-squat failure accounting is incomplete

**File:** `services/pose/poseEngine.ts`

When the bottom hold is shorter than required, the engine marks the current rep as errored but does not consistently record an error code, increment attempted reps, reduce score, or emit a coaching event.

**Impact:** Live feedback and final summary disagree about what happened.

### H8. Form state can be incorrectly reset at the start of a frame

**File:** `services/pose/poseEngine.ts`

In `READY`, the engine resets `hasFormErrorInRep`, `lastErrorMessage`, and `guidanceHint` before checking whether a new movement begins. This makes state behavior depend heavily on the exact frame where a transition occurs and can clear useful feedback immediately.

The same general pattern appears in both squat and curl paths.

### H9. Session lifecycle is not robust against unmount/restart races

**Files:** `app/pose/session.tsx`, `services/pose/index.ts`

The session screen uses a process-wide singleton engine. Cleanup calls `endSession()` whenever the screen unmounts. Rapid navigation, remounts caused by prop changes, or overlapping screens can end the active session unexpectedly.

**Impact:** Counts or summaries can belong to the wrong session, and a new session can be reset by old cleanup logic.

## Medium-Severity Flaws

### M1. `REP_COMPLETE` is effectively invisible to the UI

**File:** `services/pose/poseEngine.ts`

The engine sets `REP_COMPLETE`, but the next processing path immediately returns to `READY`. The completion state is not reliably observable as a distinct tick.

**Impact:** The UI cannot consistently show a completion phase or trigger a one-frame completion event.

### M2. Form scoring is not the documented rolling score

**File:** `services/pose/poseEngine.ts`

The current score starts at 100, subtracts fixed values on some failures, and adds one for passed reps. It is not a rolling five-rep average and does not consistently account for every attempted rep.

**Impact:** The score can be misleading and is not comparable across sessions.

### M3. Error aggregation is only partial

**File:** `services/pose/poseEngine.ts`

Some failures update `errorLog`, but other failures, including pause failures and some rejected-form paths, do not. `RepResult[]` is not retained even though the contract requires it.

**Impact:** `commonErrors` and session summaries under-report problems.

### M4. Coaching event debounce and prioritization are missing

**Files:** `services/pose/poseEngine.ts`, project engine contract

Events are pushed directly into a per-tick array. There is no general three-second duplicate suppression or priority queue. Repeated events can be emitted continuously.

**Impact:** Coaching can become noisy and repetitive, especially with jitter near a threshold.

### M5. Low visibility does not reset or safely suspend an active rep

**File:** `services/pose/poseEngine.ts`

When landmarks become unavailable, the engine returns `visibilityOk: false`, but it does not clearly transition an in-progress rep back to calibration/idle or invalidate the partial movement.

**Impact:** A user can leave the frame and return with stale FSM state, potentially completing a rep from disconnected motion.

### M6. The engine uses a single side for squat depth, not a bilateral consistency check

**File:** `services/pose/poseEngine.ts`

Even if side locking is added, one side alone can be occluded or misdetected. The engine does not compare both sides when both are sufficiently visible.

**Impact:** A single bad landmark chain can determine the entire rep.

### M7. The angle display is generic and can be wrong for the exercise

**Files:** `components/pose/SkeletonOverlay.tsx`, `store/poseStore.ts`

The UI always labels the target as `Flex Goal`, with a default target angle of 60 degrees. Squats use a target around 90 degrees, and push-up is unsupported.

**Impact:** Users receive incorrect visual guidance even when the engine angle is valid.

### M8. Exercise title handling does not cover push-up

**File:** `app/pose/session.tsx`

The title uses bicep-curl text for `bicep_curl` and otherwise displays `Squats`. A push-up session would be labeled as squats.

### M9. The skeleton overlay uses hard-coded visibility and connection assumptions

**File:** `components/pose/SkeletonOverlay.tsx`

The overlay uses a fixed `0.5` visibility threshold while the engine uses exercise-specific thresholds. It also draws all body connections regardless of the active exercise and does not indicate which landmarks actually drove the engine decision.

**Impact:** Visual skeleton quality and engine validity can disagree.

### M10. Multiple-person handling is not explicitly guarded

**Files:** `services/camera/frameProcessor.ts`, MediaPipe configuration

The callback takes one pose, but the code does not clearly configure or validate `numPoses: 1` in the application path. If multiple poses are returned, the first pose is used without a user-selection or identity rule.

**Impact:** Another person entering the frame can hijack the count.

### M11. Camera processing error handling only logs an error

**File:** `services/camera/frameProcessor.ts`

Detection errors are sent to `console.error`, but no user-visible unavailable/error state is propagated to the session UI.

**Impact:** A broken native detector can look like a counting problem to the user.

### M12. Session summary is logged but not shown or persisted

**File:** `app/pose/session.tsx`

On cleanup, the summary is sent to `console.log`. There is no summary route handoff and no persistence integration.

**Impact:** Session results are lost after the screen closes.

## Low-Severity and Release-Readiness Flaws

### L1. Automated tests and synthetic replay fixtures are absent

The engine contract requires unit tests for angle math, filtering, FSMs, form scoring, coaching debounce, and session summaries. No such test suite or mock landmark stream is present in the repository structure.

Without deterministic replay tests, the claimed rep-counting accuracy cannot be measured.

### L2. Typecheck is blocked by `tsconfig.json`

`npm run typecheck` currently fails before checking the pose files because the inherited Expo configuration conflicts with the configured `module` and `moduleResolution` options:

- `TS5095`: `bundler` requires a compatible module setting.
- `TS5109`: `moduleResolution` must be `NodeNext` when module is `NodeNext`.

This is a repository-level validation blocker, separate from pose behavior.

### L3. Native architecture/build assumptions need verification

The MediaPipe package requires React Native New Architecture. The generated Android project does not visibly confirm `newArchEnabled=true` from the current workspace search, so this must be verified before changing pose code or blaming runtime detection.

### L4. iOS model bundling is not automated

`plugins/withPoseModel.cjs` automates Android model copying but explicitly leaves iOS Xcode resource integration manual.

**Impact:** Android may work while iOS fails to load the model.

### L5. The adapter uses `any[]`

**File:** `services/camera/landmarkAdapter.ts`

The native result shape is not typed at the adapter boundary. This allowed the likely callback-shape mismatch to pass static review.

### L6. Documentation and implementation have drifted

`pose_analysis.md` contains a mixture of old findings and newer code assumptions. In particular:

- It says variation is hardcoded, but variation selection is now wired.
- It says squat only uses the left side, but current code selects by visibility.
- It says form score is always 100, but current code mutates it inconsistently.
- It describes push-up as part of the contract, while implementation supports bicep curl instead.
- It documents 5-10 Hz processing, while the implementation targets 60 Hz.

The document should be treated as an audit snapshot, not as the source of truth.

## `pose_analysis.md` Implementation Comparison

| Area | Documented expectation | Current implementation | Status |
|---|---|---|---|
| Model | MediaPipe Pose Landmarker Lite | Bundled model and native package present | Implemented, runtime unverified |
| Landmark count | 33 landmarks | Adapter requires exactly 33 | Partial |
| Callback contract | Landmark stream reaches engine | Callback property appears mismatched | Critical risk |
| Stream rate | 5-10 Hz | Approximately 60 Hz | Not implemented |
| Squat FSM | Depth, timing, pause variation | Basic FSM exists | Partial |
| Push-up FSM | Required by project docs | Missing | Not implemented |
| Bicep curl FSM | Not part of original POC contract | Implemented | Scope drift |
| Form checks | Posture and alignment checks | Mostly absent | Not implemented |
| Visibility filter | Reject low-confidence points | Present but missing confidence defaults to 1.0 | Unsafe partial |
| Side selection | Stable anatomical chain | Visibility selection every frame | Unsafe partial |
| Form score | Rolling score | Ad hoc fixed increments/decrements | Not compliant |
| Error summary | Common error aggregation | Partial error map | Partial |
| Coaching | Debounced/prioritized events | Per-tick event arrays | Not implemented |
| Variations | User-selectable thresholds | UI handoff works for exposed exercises | Implemented for squat/curl |
| Summary | Complete and handed off | Only logged on cleanup | Not implemented |
| Tests | Deterministic replay and >=85% coverage | No pose test suite found | Not implemented |
| Device benchmark | Run once and tier device | No implementation found | Not implemented |
| Audio coaching | Native ducking and playback | No implementation found | Not implemented |

## Dependency and Architecture Assessment

### Do not change dependencies yet

The current dependency set appears sufficient for the intended architecture:

- Expo SDK 57
- React Native 0.86
- Vision Camera 4.7.3
- MediaPipe pose detection 0.4.0
- Worklets
- Skia
- Zustand

The first work should verify and repair the existing integration contract. Adding another pose library or replacing MediaPipe before proving the callback, model-loading, orientation, and New Architecture paths would create unnecessary dependency risk.

### Dependency/build checks required before implementation

1. Confirm the exact runtime callback object from `react-native-mediapipe-posedetection`.
2. Confirm `newArchEnabled=true` for Android and the equivalent iOS setup.
3. Confirm the model is packaged and loadable on the target Android device.
4. Confirm iOS model resource bundling before claiming iOS support.
5. Confirm the camera orientation/mirror behavior with a known-pose replay.
6. Repair the TypeScript configuration enough to run focused type checks.

## Recommended Fix Order

1. Verify and correct the MediaPipe callback shape.
2. Add typed native-result and adapter contracts.
3. Add a deterministic landmark replay harness and angle/FSM unit tests.
4. Decide the supported exercise scope: push-up or bicep curl.
5. Add temporal filtering, visibility handling, and stable side selection.
6. Implement the selected exercise FSM and all required form checks.
7. Add strict exercise/variation validation.
8. Normalize rep-result, error, score, and coaching-event accounting.
9. Validate orientation and overlay coordinates on a physical device.
10. Complete summary handoff, device benchmark, and release checks.

## Bottom Line

The current feature should not be judged by threshold tuning alone. First prove that the correct 33-landmark frame reaches the engine. Then make the engine deterministic under noisy input with replay tests. Only after those two gates pass should thresholds, posture heuristics, and device-specific tuning be adjusted.
