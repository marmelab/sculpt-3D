## Project Overview — 3D Sculpting Web App
This application is a web-based 3D modeling and sculpting tool (ZBrush-like) built with React and React Three Fiber. The user works on a modeling canvas with horizon grid and 3D axes, can add primitive shapes using click‑and‑drag placement, switch tools (select / add primitive / shaded / mesh / move / scale / add-matter / subtract-matter / push), sculpt meshes by deforming geometry continuously while the mouse is held, and use an oriented brush preview. Dynamic, adaptive edge-based subdivision runs during sculpting to keep triangle sizes within threshold without creating T-junctions or holes. The selected object uses an outline and selectable render mode: shaded (default) or mesh (wireframe-only with proper hidden-line removal). Object-specific context actions and properties appear in a sidebar.

This specification reflects the current, integrated state including recent pivots: symmetry in object-local space with visual planes, stronger Push tool defaults, a global scene-based undo/redo system, extraction of sculpting logic into pure, testable headless functions with unit tests (including subdivision), robust symmetric subdivision and post-processing to ensure topological symmetry, versioned in-place geometry management (ref + version counter), improved primitive generation (icosphere + adaptive initial topology), centralized camera/rotation control for mobile, vertex merging to avoid seams, TDD coverage of subdivision and topology invariants, tuning of tool strengths/behaviors, and recent UX and mobile-interaction refinements and rollbacks following usability testing.

## Main Goals
Enable intuitive polygonal sculpting and basic modeling workflows with precise placement, camera-safe interactions, predictable adaptive subdivision, robust symmetry tools, reliable undo/redo semantics across the scene, and testable core logic. Key goals are:
- Click‑and‑drag primitive placement (center, size, full 3D orientation) with Shift-constrain for straight shapes.
- Continuous sculpting tools that displace or move vertices using averaged normals or camera-derived push directions.
- Symmetric sculpting in object-local space with visual planes and replication strategy that ensures exact replication of computed edits rather than recomputing them on mirrors.
- Adaptive edge-based subdivision that preserves mesh integrity (no T-junctions, no holes), runs deterministically and symmetrically, and is covered by unit tests.
- Global, scene-level undo/redo that records complete scene snapshots per user gesture (creation, deletion, transform, sculpt stroke), independent of current selection.
- In-UI, in-place mutable geometry updates stored in refs for performance, with a numeric version counter in React state used to force re-renders and detect race conditions (single source of truth for concurrency control).
- Headless, pure sculpting/subdivision functions to allow fast unit tests that reproduce UI scenarios and catch cases where subdivision perturbs tools.
- Mobile usability: touch-friendly placement and sculpting gestures, compact and grouped UI for small screens, optional dialogs for full tool controls on mobile, and a reliable, testable mobile rotation affordance that does not conflict with tools.

## Features

Features are grouped by functionality. Each feature description explains the user problem it solves, what it does, how to use it, user inputs and expected outputs, business/validation rules, limits and error cases, and acceptance notes expressed as concise prose.

### Modeling Canvas & Camera
Provides the 3D workspace and camera navigation without interfering with tool interactions. The canvas displays a horizon grid and axes. Orbit and zoom use standard mouse/touch gestures; middle mouse or configurable control rotates by default on desktop. Mobile-specific rotation affordances are provided without interfering with primary touch tool gestures.

User input: left click (primary) on empty background rotates (on desktop) when allowed; middle-drag or touch gestures also rotate/zoom per device. Left clicks (or taps) on objects are routed to tool handlers. Output: updated camera view.

Business rules: camera actions must not trigger tool actions. Raycasting is performed only on click/mousedown to determine if the pointer hit an object (explicit performance requirement: do not raycast on mousemove). A centralized Scene-level mousedown handler runs in the capture phase to decide whether a click should enable OrbitControls for that event; OrbitControls.enabled is adjusted transiently to let the native OrbitControls process rotation for clicks on empty space while preventing left-click rotation when the click hits an object. Camera controls are suspended while the user is actively placing a primitive, transforming an object (move/scale), or using sculpt-like tools so tool drag semantics are not masked by orbiting. The raycast check runs only on mousedown (not on mousemove) for performance.

Mobile rotation: because touch usage collides with sculpt gestures, provide a separate, dedicated rotation widget (bottom 3‑axes widget) and an optional "View" activation flow. Any attempt to change OrbitControls behavior globally for all touch events is avoided to prevent pointer-capture and synthetic-event recursion issues. Implementation choices include: a small persistent rotation widget that on pointer-drag temporarily enables rotation for the gesture, a "view" tool that can be explicitly selected, and explicit pointer capture/release only within controlled widget handlers. The app must avoid synthetic event loops and guard against releasePointerCapture errors by correctly managing pointer ids and using event.isTrusted checks where synthetic events are dispatched in fallback flows. Recent exploratory implementations that modified multiple components and introduced bugs must be reverted if they cause user-visible regressions.

Acceptance: Clicking empty space rotates; clicking objects invokes tools and does not rotate. On mobile, dragging the bottom widget rotates the view reliably without interfering with sculpt gestures. Raycasts only run on click/mousedown, never on pointermove/mousemove for hover detection.

### Tooling & Toolbar (Select / Add Primitive / Move / Scale / Add / Subtract / Push / View)
Solves switching between object selection, placement, transforms and sculpt operations. Use the toolbar to pick the active tool.

What it does/how to use: Select picks objects; Add Primitive enters click‑and‑drag placement: first click sets center (on ground or surface), drag sets size and orientation; Shift constrains rotation. Move translates selected object in camera-relative space; Scale scales selected object uniformly or per axis. Add/Subtract apply additive/subtractive sculpting by displacing vertices along averaged normals; Push applies drag-derived 3D displacements with progressive falloff. A dedicated View tool exists (optional) to explicitly enable rotation when selected; a mobile rotation widget can temporarily switch to view behavior for the gesture and then restore the previous tool.

User input/output: click/drag + modifiers. Output: new primitives added, selected object transforms updated, or mesh geometry modified.

Business rules and validation: Move/Scale/Add/Subtract/Push require selection; their toolbar buttons are disabled when no object selected (except Add Primitive and View). Add Primitive center must resolve a hit; otherwise place on ground plane. Scale enforces configurable min/max. Continuous operations capture a starting geometryVersion; all in-place edits increment the SceneObject version on commit. If a commit returns a version jump greater than +1, that indicates a concurrent edit and the new edit is discarded by default (race-detection policy). Tool-specific brush controls appear during sculpting.

Limits and error cases: Transform inputs validated numerically. Continuous sculpting rate is throttled to avoid main-thread starvation; subdivision and topology edits are bounded by per-frame and per-object budgets. If no valid raycast hit for sculpt tool when required by tool, sculpt is a no-op.

Mobile toolbar grouping and dropdowns: On narrow viewports, sculpting-related buttons are grouped in a dropdown to avoid overflow. When a tool switch hides secondary toolbars (like shape selectors), those secondary toolbars must auto-close on finalization of the placement gesture; lingering secondary panels after placement are considered a bug and must be closed by a change in currentTool state.

Acceptance: Tools apply continuously while mouse held; brush preview matches deformation and tools are responsive without unexpected rotation.

### Placement (Click‑and‑drag center/size/orientation) — touch-enabled
Allows precise placement of primitives with full orientation and size on desktop and touch devices. First pointer down defines center; dragging sets size and quaternion rotation derived from world-space drag vector (unless Shift or touch modifier). Mesh resolution scales with final size so larger objects start with higher topology.

User input/output: Pointer down (center) → preview follows drag with rotation and scale; pointer up finalizes SceneObject creation and mesh resolution proportional to final size. On mobile, touch gestures for placement must work without being intercepted by global rotation gestures; tool activation and pointer capture should ensure consistent behavior.

Business rules: Mesh resolution = baseResolution * sizeScaleFactor, configurable. Orientation defaults to identity when drag below threshold; Shift locks rotation (desktop). If no raycast, place on ground plane at pointer ray distance and respect snapping if enabled. Primitive generation now uses an icosphere/polyhedron generator by default to avoid UV pole artifacts and computes target vertex counts based on object size (logarithmic/edge-length target). PrimitiveFactory merges duplicate vertices at creation time to ensure watertight topology.

Validation: enforce min/max scale. Acceptance: Placement yields expected center, size, orientation, and appropriate mesh resolution; new primitives are watertight and have pre-merged vertices. Touch placement works reliably on mobile (no accidental rotation, no dropped gestures).

### Scene Objects & Rendering / Selection Outline / Render Modes
Each SceneObject encapsulates geometry, adjacency/topology metadata, a mutable geometry ref (not React state), and a geometryVersion numeric state used to trigger re-renders and for concurrency control.

What it does: Shaded mode renders the object with surface shading using the same BufferGeometry manipulated by sculpting and subdivision. Mesh mode shows a wireframe-only hidden-line-correct rendering using a depth pre-pass and wireframe pass. Both modes represent the same underlying geometry. SceneObject provides an onGeometryUpdate(versioned update) callback to accept in-place geometry edits from sculpting hooks; updates include startingVersion and new vertex counts so SceneObject can validate before accepting.

User input/output: sidebar toggles render mode; switching takes effect immediately. Rendering mode toggles do not create duplicate geometry; they reuse underlying BufferGeometry. SceneObject ensures that mobile settings UI can toggle render mode reliably.

Business rules: Only the geometry ref is mutated; React re-renders are forced by incrementing geometryVersion state. onGeometryUpdate callbacks must pass the startingVersion for race detection. Mesh mode uses the same BufferGeometry and must hide occluded edges via depth technique. Acceptance: Shaded and Mesh views remain synchronized; toggles do not crash.

### Shaded Rendering Consistency (avoid polyhedron look)
Problem solved: avoid faceted look at typical camera distances.

What it does: Primitive generators adapt topology based on final size with icosphere (icosahedron-subdivision) generator and target edge length heuristics. After topology changes (primitive creation or subdivision), normals are recomputed. Sphere primitive generator computes subdivisions targeting small edge length so large spheres begin with high vertex counts to keep them smooth; the generator respects per-object vertex budget.

Business rules: Primitive creation computes adaptive resolution proportional to object size to avoid coarse shading. Subdivision is driven by sculpt operations; shading remains accurate by recomputing normals after edits. Acceptance: Large spheres and other primitives appear smooth without pole artifacts.

### Sculpt Tools (Add / Subtract / Push) — continuous vertex deformation & push behavior
Solves realistic, continuous sculpting. Add/Subtract displace vertices along averaged normals. Push moves vertices along a 3D drag vector anchored to the initial hit depth.

What it does/how to use: While pointer held, raycast hit face identifies a brush center. The engine computes affected vertices through adjacency and applies displacement per tool: Add/Subtract apply averaged normal displacement with radial falloff; Push computes 3D movement from unprojected pointer deltas anchored at hit depth and displaces nearby vertices with falloff. Adaptive edge-based subdivision runs before displacement where needed. Symmetry (if active) replicates edits by replicating exact vertex/index deltas rather than recomputing mirrored edits. A headless sculpting engine provides pure functions for stroke application to allow unit testing (including subdivision).

User input/output: brushSize and brushStrength controls, pointer down + move apply deformation; output is updated BufferGeometry in-place and the SceneObject version incremented on commit.

Business rules and validation: Affected vertex selection uses adjacency rings. Displacement direction for Add/Subtract uses averaged normals. Brush falloff is radial and configurable. Continuous application rate is throttled (updates per second). Subdivision runs synchronously in the affected neighborhood before displacement on edges exceeding maxEdgeLength. Per-object mutation uses the versioned lock: sculpt captures startingVersion; when the sculpt commit results in a new geometry (or modified positions) SceneObject should accept it only if currentVersion equals startingVersion; the component then increments version +1 to force render. If the version check fails (i.e., mismatch indicating concurrent edit), the new edit is discarded by default and a warning logged. Max displacement per frame prevents degenerate triangles; post-update integrity checks run and revert the last update on failure.

Symmetry rules: Symmetry operates in object-local space. For each stroke, the engine computes symmetry points in object-local coordinates and first computes the edit on the primary point(s), then replicates the exact computed vertex and index changes to the mirrors. Subdivision is performed symmetrically by considering unified neighborhoods across symmetry points; a deterministic post-processing pass (ensureGeometrySymmetry) repairs missing mirror vertices/triangles.

Limits and errors: If no valid hit, sculpt is no-op. Brush preview matches actual deformation; preview sampling uses the same adjacency logic but is throttled for performance. Acceptance: Tools produce predictable continuous edits; symmetry reliably mirrors edits, and subdivision preserves topology without T-junctions.

### Adaptive Edge-based Subdivision (no T-junctions, hole-free)
Solves inability to sculpt fine detail in coarse areas while preventing topology corruption.

What it does: Before displacement, the subdivision module examines edges in a bounded radius around the brush centers (including all symmetry points). Any edge longer than maxEdgeLength (relative to brush size and desired detail) is marked for splitting. Subdivision builds an edge->triangles adjacency map, marks edges, and then ensures that all triangles sharing marked edges are processed in a way that produces consistent outputs for every triangle regardless of how the edge was originally marked. The algorithm first ensures geometry has consistent indexing (PrimitiveFactory merges duplicate vertices at creation time to avoid seams) and contains robust fallback merging for externally created geometries in unit tests. Splits use edge-keying and closure propagation so that any neighbor triangles that share split edges are also updated, preventing T-junctions and holes. Subdivision is bounded by per-frame budgets and overall per-object vertex/index budgets and is symmetric-aware.

User input/output: automatic during sculpt flows; users can tune maxEdgeLength and subdivision aggressiveness in settings. Output: refined local mesh without T-junctions; adjacency maps are updated synchronously within the per-object mutation window.

Business rules: Edges selected for splitting form a closure region to maintain consistent topology. Splits are propagated to neighbors where necessary. Subdivision per frame is clamped by budget. If exceeding global vertex/index budget, subdivision is skipped and user warned. A TDD suite validates watertightness: edges must be shared by exactly two triangles and no T-junctions (vertex-on-edge without neighbor split) should exist.

Validation and limits: Subdivision radius couples to brush radius. Acceptance: local subdivision, no T-junctions, respects budgets, and symmetric subdivision produces mirrored topology. Subdivision and topology fixes are covered by unit tests that reproduce scenarios across sphere/cube/cylinder/cone/torus and simple quads.

### Brush Preview & Surface-oriented Projection
Solves need for low-latency accurate visual cues matching sculpt target.

What it does: A projected circular brush on the surface oriented by averaged normal from the hit face and limited adjacency rings. Preview updates at pointer move frequency but is throttled. For Push, preview indicates direction while dragging. When symmetry is enabled, visual symmetry planes render in object local space and mirrored previews show per-symmetry-point overlays. BrushPreview provides a deterministic “isHovering” state used by centralized Scene mousedown logic to avoid rotating when clicking an object.

Business rules: Preview sampling uses only local adjacency rings. Throttling is modest to reduce jitter. Preview uses identical vertex selection logic as sculpting. Acceptance: Preview matches deformation area and orientation; preview hover flag is authoritative for click handling as needed.

### Raycasting & Adjacency Selection (performance optimization)
Solves expensive global scans by using local neighborhoods.

What it does: Raycast returns face index and barycentric coordinates. Mesh adjacency (vertex->faces, face->faces) is precomputed at creation or lazily updated and used to gather nearby vertices/edges. After topology edits, adjacency updates within the per-object version lock. Scene-level mousedown handler uses a fast raycast against tracked mesh refs only on click (capture phase) to decide whether click hits an object or background. Raycasts must not be performed on mousemove/pointermove.

Business rules: Adjacency stored per-mesh and updated inside the per-object mutation window after topology changes. Avoid iterating the full vertex buffer per pointer event. Acceptance: Adjacency updates complete before dependent sculpt operations run.

### Object Sidebar (context toolbar)
Provides context toolbar for the selected object with render mode toggle, vertex count, transform controls, delete and shortcuts.

What it does/how to use: When an object is selected, the sidebar shows Shaded/Mesh toggle, live vertex count, numeric position/rotation/scale inputs, and Delete. Render mode toggle switches between Shaded and Mesh rendering of the same underlying geometry.

Business rules: Transform inputs validate numeric ranges and update object state immediately. Deleting an object releases adjacency and geometry memory and creates a global undo entry for deletion. Move/Scale toolbar buttons disabled when no selection. Acceptance: Vertex count reflects BufferGeometry length after topology edits. Mobile settings panel includes the same render mode toggle.

### Undo / History (Global Scene Undo/Redo)
Problem solved: revert unintended edits, including creations and deletions, consistently across selection changes.

What it does and how to use: A global undo/redo system records the full scene snapshot (all objects, their geometry buffers/metadata, selection and object versions) per discrete user gesture. Keyboard shortcuts (Ctrl+Z / Ctrl+Shift+Z) and UI buttons expose undo/redo. Each continuous gesture—placement (single), transform drag (batched), sculpt press-drag (batched), deletion—is recorded as a single undo entry. Undo reverts the complete scene snapshot so deletions can be restored even if selection changed. Redo reapplies the snapshot.

User input/output: clicking Undo reverts the last operation; Redo reapplies. Output: scene objects, geometry, selection, and vertex counts reflect the undone state.

Business rules: The undo system is scene-global, not tied to a particular object. Each entry stores object geometries and adjacency so subsequent sculpting remains consistent. Undo stack depth and memory caps are configurable; when capacity exceeded, oldest entries are dropped with notification. Continuous gestures are recorded on gesture completion (pointer up). Per-frame topology changes (e.g., subdivision during a stroke) are included within the same undo entry so reverting a sculpt stroke restores prior topology and adjacency.

Validation and limits: Undo must restore BufferGeometry and adjacency structures as recorded in snapshot. Acceptance: Undo reliably undoes single gestures, deletions are undoable, and selection changes do not corrupt undo/redo semantics.

### Symmetry (Object-local, Visual Planes, Replicated Edits)
Problem solved: symmetric sculpting for facial/human feature workflows; assure symmetric topology even during subdivision.

What it does/how to use: Symmetry axes X/Y/Z toggles appear in SculptingControls and have keyboard shortcuts (X/Y/Z). Symmetry planes are displayed in object-local coordinates and follow object transforms. When symmetry active, the engine computes symmetry points (object-local) for the brush center and performs the edit on the primary point(s) and then replicates the computed vertex/index changes to mirror locations instead of recomputing edits there. Subdivision marking considers unified neighborhoods including all symmetry points. A post-processing pass (ensureGeometrySymmetry) verifies and corrects missing mirror vertices or triangles.

Business rules: Symmetry operates in object-local space. Mirror replication takes the exact computed deltas from the main side and applies them transformed to mirrored vertices to avoid divergence. Visual planes render in local object transform and update with object edits. Acceptance: symmetry reliably produces symmetric mesh and displacement, even with subdivision.

### Headless Sculpting Engine & Test Infrastructure (TDD emphasis)
Problem solved: slow UI testing and hard-to-reproduce symmetry/subdivision bugs.

What it does: Core 3D manipulation logic (tools, adjacency-aware vertex selection, subdivision, symmetry replication, deformation application) is extracted into pure, headless functions in sculptingEngine and symmetricSubdivision modules. These functions support both in-place mutable operations (for UI performance) and clone-and-return immutable flows (for tests) via a cloneGeometry flag. A comprehensive vitest test suite exercises sculpt strokes (including subdivision), symmetry, watertightness, and topology invariants. Utilities include verifyWatertightMesh() and findTJunctions() to assert every edge is shared by exactly two triangles and no vertex lies on an edge without neighbor subdivision.

Developer use: Run unit tests to reproduce hard UI bugs deterministically. Tests simulate continuous strokes, subdivision, adjacency updates, vertex merging, symmetry, and race scenarios (version jumps). Tests are used to detect intermittent "blinking" or revert regressions by reproducing sequences of in-place edits and verifying final topology.

Business rules: Mutable operations use geometry refs and must be guarded by the SceneObject versioning mechanism in the UI to avoid race conditions; tests use immutable clones. All subdivision logic is exercised by tests so regressions are caught early. Acceptance: Tests reproduce UI bug scenarios and validate fixes before merging.

### Geometry Versioning & In-place Updates (Race Condition & Re-rendering Strategy)
Problem solved: UI hangs, blinking, and edits being immediately canceled due to race conditions between subdivision and continuous edits or between multiple simultaneous edits.

What it does: Each SceneObject maintains geometryRef (the BufferGeometry object) and a numeric geometryVersion stored in React state. Sculpting and other mutating operations modify the geometry in place for performance and then call onGeometryUpdate callback with the startingVersion and optional geometry reference changes. SceneObject accepts updates only when currentVersion equals startingVersion; it then assigns the mesh.geometry ref as appropriate and increments geometryVersion by +1 to force a React re-render (or applies policy to merge). If a commit sees a version jump greater than +1, that indicates overlapping edits and the second edit is discarded by default (or queued with explicit policy). This replaces previous isProcessing boolean locks. Subdivision and adjacency updates occur synchronously within the mutation window before the version is validated and incremented. Undo records include geometry clones or diffs sufficient to restore previous versions.

Business rules: Edits stamp the object's startingVersion and validate on commit. If commit sees a version mismatch, the edit is discarded and optionally a user notification/log entry is created. Version increments must be monotonic. Acceptance: UI no longer blinks or reverts inadvertently; subdivision changes persist after stroke completion; race conditions reliably detected.

### Primitive Generation & Adaptive Initial Topology (Icosphere + vertex merging)
Problem solved: large primitives appear faceted because initial topology was insufficient; Three.js geometry seam duplication causes T-junctions under selective subdivision.

What it does: Primitive generators choose an initial topology count based on final object size and a configurable density factor. For spheres, an icosahedron-based generator (THREE.IcosahedronGeometry with computed subdivisions) is used to avoid pole artifacts. The generator computes target vertex counts by estimating initial edge length and halving it per subdivision until the target edge length is reached (log2). Upper limits exist to prevent runaway memory usage. Immediately after creation, primitives are configured: position BufferAttribute set to dynamic usage, normals and bounds computed, and duplicate vertices merged (epsilon = 0.001 matching symmetry tolerance) to produce a watertight indexed mesh. The merge is done once at creation for efficiency; subdivision functions include a fallback merge only for externally created geometries or tests.

Business rules and validation: The generator must not exceed per-object vertex budget; if the computed target exceeds budget, it caps and warns user. Mesh merging uses epsilon matching; merging is deterministic. Acceptance: primitives are created smooth at intended size and are watertight, preventing T-junctions during early sculpting.

### T-junction & Watertightness Guards (Subdivision correctness)
Problem solved: T-junctions/holes appearing when a triangle subdivides an edge while a neighbor didn't.

What it does: Subdivision algorithm creates a complete edge->triangles adjacency map and computes a set of edges to split based on geometry-edge length relative to threshold. The subdivision pass ensures that for every triangle processed, it uses the presence of midpoints (edgeMidpoints map) rather than assuming only triangles that had marked edges will be updated. Triangles that would be inconsistent due to a neighbor-created midpoint are handled by detecting that edgeMidpoints contains the midpoint and updating that triangle's output pattern accordingly. Tests (simple quad, real-world primitives) verify every edge is shared by exactly two faces after subdivision and that no vertex projects onto an edge (T-junction). Iterative propagation logic ensures closure expansion is correct without over-marking.

Business rules: Vertex merging at creation eliminates seam duplicates; subdivision must update both triangles sharing a split edge to include the new midpoint. EdgeMidpoints are computed globally before reconstructing triangle outputs, making per-triangle decisions consistent. Acceptance: Unit tests for quad, sphere, box, cylinder, cone, torus pass; the UI shows no tearing during sculpting.

### Logging, Console Noise & TypeScript Correctness
All debug logging gated behind developer toggle; production logs only warnings/errors. TypeScript correctness enforced; builds fail on type errors.

Business rules: No stray console.log in production. Acceptance: clean console in normal usage; TypeScript build passes.

### Mobile UX & UI Optimizations (new & updated)
Problem solved: mobile devices have limited screen space and touch interactions collide with tool gestures; sculpting mode previously displayed too many dialogs and prevented interaction.

What it does: Introduces a mobile-specific UI layer: a compact top toolbar with grouped dropdowns for sculpting tools, an optional modal/dialog to expose full tool controls (brush size, brush strength, symmetry toggles), a collapsible settings panel with render mode toggle (shaded/mesh) for the selected object, and a bottom rotation widget (3‑axes) to handle rotation gestures without interfering with sculpting. The mobile UI uses a useIsMobile hook to control responsiveness and allow feature gating.

User input/output: on mobile, tapping the compact sculpt group opens a small dropdown with tool items; a separate "Tool Controls" dialog reveals brush size/strength/symmetry. The settings panel includes a Render Mode selector. The rotation widget handles pointerdown + drag to rotate the camera and temporarily enables rotation behavior only for that widget's gesture.

Business rules: The sculpt controls dialog is optional and user-toggleable to avoid obstructing the canvas. Grouped toolbar items collapse on narrow widths; secondary toolbars (e.g., primitive subtype selectors) auto-close when the tool completes. The rotation widget must manage pointer capture correctly: capture pointer id on pointerdown, ignore spurious pointer events (guard releasePointerCapture calls), and ensure pointer events are not re-emitted to the canvas as synthetic events to avoid recursive handlers. Recent experimental code that introduced recursion or pointerCapture DOM exceptions is reverted; the rotation widget must be implemented in a way that avoids these pitfalls (explicit pointer id storing, guarded release calls, and no synthetic dispatch unless strictly necessary and flagged by event.isTrusted checks).

Validation: mobile toolbar groups must not hide essential controls; tool controls dialog must show the three requested controls: brush size, brush strength, symmetry. Acceptance: mobile users can place and drag new objects by touch, open tool controls dialog for sculpting, use grouped toolbar dropdowns, and rotate the view via the bottom widget reliably without errors in the console.

## Overview of Technical Decisions (high-level, one page)
Responsibilities are organized around SceneObject units that encapsulate a mutable BufferGeometry reference and a geometryVersion counter, adjacency/topology metadata, and a per-object concurrency contract implemented via version stamping. Core sculpting and subdivision logic is extracted into headless modules providing pure functions (configurable to mutate in-place or return clones) to enable deterministic unit testing and easier diagnosis of regressions. Primitive generation prefers icosphere/polyhedron generators and computes initial topology target by estimating target edge length based on object size; merged vertices at creation yield watertight indexed meshes, avoiding seam duplication that causes T-junctions upon selective subdivision. Subdivision is edge-keyed and adjacency-aware, performed on unified neighborhoods that include all symmetry points and using a two-phase approach: compute edge midpoints for the full set, then recompose triangle outputs based on which midpoints exist (ensuring neighbor agreement), with per-frame budgets to bound work. Symmetry is computed in object-local space and mirror edits replicate exact computed vertex/index deltas. The UI uses in-place geometry updates for performance but relies on the numeric geometryVersion to force re-renders and detect concurrent edits: edits stamp the startingVersion and SceneObject accepts only if currentVersion matches; the accepted update increments version by +1. A centralized Scene-level mousedown (capture) handler decides whether a left click should be treated as camera rotation by raycasting once per click, avoiding per-tool toggles and per-frame raycasting. Mobile interaction strategy favors explicit, localized widgets and dialogs to avoid global control toggling that led to pointer-capture recursion and synthetic event issues in prior attempts. A comprehensive TDD suite (vitest) covers sculpting behavior, symmetric subdivision, watertightness, and race scenarios. The reasoning behind these choices balances performance (in-place updates, refs, single-spot camera control), determinism and correctness (pure headless functions, version stamping), and user expectations (no tearing, no blinking, predictable undo/redo). Recent decisions include reverting any experimental multi-component event-phase changes that caused instability and instead implementing a dedicated, self-contained rotation widget and explicit mobile UI controls.

Key Challenges & Solutions:
- T-junctions when neighbors disagree on edge splits: solved by precomputing midpoints for all marked edges and reconstructing triangles based on midpoint presence, plus vertex-merging at creation.
- Intermittent subdivision “blinking” / reverts: solved by switching geometry to ref-based in-place mutations and using geometryVersion for re-rendering and race detection rather than a boolean isProcessing.
- Symmetry divergence during subdivision: solved by computing edits on primary side and replicating exact vertex/index deltas to mirrors; symmetric subdivision marks unified neighborhoods and runs post-processing ensureGeometrySymmetry.
- Performance vs testability: mutable in-UI updates with a cloneGeometry flag for tests enable both fast UI and deterministic unit tests.
- Mobile rotation vs tool input: avoid global OrbitControls hacks that caused pointer capture recursion and synthetic-event loops. Instead, provide a dedicated bottom rotation widget (or explicit View tool selection) that manages its own pointer lifecycle and temporarily enables rotation behavior for the duration of the widget gesture, ensuring pointer ids and release calls are managed safely.
- Unintended code changes and regressions: when a change creates a failing interaction (e.g., left-click on object still rotates or mobile rotation causing console errors), revert to the last committed stable state and reintroduce fixes behind test coverage and incremental, validated changes. All event-phase handling must be deterministic and documented to prevent capture/bubble ordering conflicts.

This document focuses on user-facing features and business logic, capturing the current integrated state, recent pivots, and the latest user-driven constraints: do not raycast on mousemove, ensure left-clicks on objects do not trigger rotation, provide mobile UX improvements including compact grouped toolbars and an optional sculpt-control dialog, reliably support touch-based placement and rotation (via a rotation widget or explicit tool), and maintain the same robust sculpt/subdivision/symmetry semantics and test coverage.
