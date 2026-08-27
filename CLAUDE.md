# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

KnitSim is a browser app for designing knitting patterns and simulating the resulting fabric's physics. All code lives under `knitviz-frontend/`: a Vue 3 + Vite frontend, plus a C++ simulation core that compiles to WebAssembly and runs entirely client-side (no backend/server component).

## Commands

All commands run from `knitviz-frontend/`.

```sh
npm i -D            # install JS deps
npm run dev          # Vite dev server (localhost:5173)
npm run build        # type-check (vue-tsc) + production build
npm run type-check    # vue-tsc --build, no emit
npm run preview       # preview a production build
```

### Building the WASM simulation module

The compiled sim (`knitsim-lib.js` / `knitsim-lib.d.ts`) is gitignored and **not checked in** — `src/knitgraph/sim/` only has a `.keep` file until you build it. Without it, the app loads but anything that touches `KnitSimModule` (3D preview, physics simulation) throws.

Linux/macOS (requires Emscripten SDK + Eigen 3.4 on `PATH`/`Eigen3_DIR`):
```sh
./build-wasm.sh
```
This runs `emcmake cmake . -B dist` + `emmake make -C dist`, then copies `knitsim-lib.js`/`.d.ts` into `src/knitgraph/sim/`.

Windows notes (no native `make`/homebrew Eigen path available):
- `find_package(Eigen3 3.4 REQUIRED CONFIG ...)` in `CMakeLists.txt` needs an **Eigen3 config exactly matching major version 3** — a newer Eigen (e.g. vcpkg's default, which is 5.x) will be rejected. Point `-DEigen3_DIR` at an actual Eigen 3.4.0 install/config, not just any Eigen3 package.
- Pass `-G Ninja` to `emcmake cmake` (there's no `make.exe`); build with `emmake cmake --build dist` instead of `emmake make`.
- `cmake/wasm.cmake`'s `--emit-tsd=knitsim-lib.d.ts` linker flag must use `=`, not a space + quoted string — `--emit-tsd 'file.d.ts'` was previously being passed through literally on Windows/Ninja, producing a file named `'knitsim-lib.d.ts'` (with quote characters in the name).

Docker (builds WASM + frontend together, no local toolchain needed):
```sh
docker build . -t knitviz-frontend
docker run -it -p 16001:80 knitviz-frontend
```

There is no JS/C++ test suite in this repo currently — verification is via `type-check`/`build` succeeding and manual exercising of the app.

## Architecture

### Data flow: four editors → one graph → 3D/physics

The app has four ways to author a pattern (tabs in `EditorsView.vue`: **Code**, **Grid**, **Visual** (Blockly), **Node**), but they all converge on a single interchange format:

```
editor-specific state  →  EditorAdapter.createSnapshot()  →  GraphSnapshot  →  globalEditorStore (Pinia)
                                                                                        │
                                                              snapshotToGraph()         ▼
                                                                        KnitGraph  →  KnitGraph3D  →  PatternViz3D (three.js + WASM)
```

- **`src/components/editor/adapters/*Adapter.ts`** (`codeAdapter`, `gridAdapter`, `visualAdapter`) each implement `EditorAdapter` (`adapterInterface.ts`): convert their editor's native state into a `KnitGraph`, then serialize via `graphToSnapshot()` into a plain-data `GraphSnapshot` (`src/knitgraph/snapshot.ts`). The **Node editor** instead patches individual nodes directly into the stored snapshot (`applyNodeSnapshots` in the store).
- **`src/stores/globalEditorStore.ts`** (Pinia) holds the current `GraphSnapshot` as `state.graph` plus each editor's own persisted state (so switching tabs doesn't lose work), and a `revision` counter that bumps on every graph change.
- **`src/knitgraph/snapshot.ts`**: `graphToSnapshot`/`snapshotToGraph` convert between the live `KnitGraph` (with node object references, e.g. `previous_node`) and the flat, serializable `GraphSnapshot` (IDs only) used for storage/cloning.
- **`src/components/editor/VizRenderer.vue`** turns a `GraphSnapshot` back into a `KnitGraph` and constructs a `PatternViz3D` to render/simulate it. `EditorsView.vue` is the page that wires the editor tabs, the store, and `VizRenderer` together, and owns the "Generate from X" / "Show Preview" / "Start Simulation" buttons — **generating a graph (`store.state.graph.nodes.length > 0`) is a prerequisite for Show Preview to do anything.**

### The knitting graph model (`src/knitgraph/index.ts`)

- `KnitNode`: one stitch. Key fields: `row_number`/`col_number`, `type` (`KnitNodeType`: KNIT/PURL/SLIP/YARN_OVER/INCREASE/DECREASE/BIND_OFF/CAST_ON), `side` (`KnitSide.RIGHT`/`WRONG` — which face of the fabric, not left/right position), `start_of_row`, `previous_node` (row-wise predecessor), `yarnSpec` (color/weight).
- `KnitEdge`: connects two node IDs with a `KnitEdgeDirection` — `ROW` (stitch-to-next-stitch in the same row) or `COLUMN` (stitch-to-the-stitch-below, i.e. row-to-row). `KnitGraph.row()`/`beginOfRow()`/`traverseGraph()` walk these edges to answer "what row is this node in" / "find the node N stitches away in direction X" type questions (used for increase/decrease and yarn-over stitch construction).
- `KnittingState`: a small stateful builder — `cast_on(n, mode)`, `knit(n, type)`, `color(...)`, `end_row()` — that appends nodes/edges to a `KnitGraph` as it's called, tracking current row/col/side. This is the same API surface all four editors ultimately drive (either directly, like `gridAdapter`, or via generated code).
- **`KnitGraph.execute(code: string)`** runs user/sample pattern code (see `src/stores/samples/codesamples.ts`) via `new Function(code)` with `this` bound to a `KnittingState` — so pattern scripts are literally JS calling `this.cast_on(...)`, `this.knit(...)`, `this.end_row()`. There is no sandboxing; this code runs with full page privileges.

### 3D rendering & simulation (`src/knitgraph/3d/`)

- **`graph.ts` (`KnitGraph3D`)**: extends `KnitGraph` with `KnitNode3D` nodes. `initGraphWASM(cfg)` translates the JS graph into the WASM module's `KnitGraphC` + `KnitSim` (mapping JS enums like `KnitSide.RIGHT` to WASM enums like `KnitSimModule.KnitSideC.KnitSideC_RIGHT` by string interpolation — enum names must stay in sync between `index.ts` and `wasm.cpp`). `step(time)` advances the physics sim one tick and pulls positions back via `syncGraphWASM()`. `computeHeuristics()` does the initial (non-physics) layout + knit-path computation used for the static preview.
- **`node.ts` (`KnitNode3D`)**: per-node three.js state (position, mesh/curve for non-instanced nodes, normal/force debug arrows) plus `preRender(viz)`, called every frame to sync visibility flags from the parent `PatternViz3D`'s GUI toggles.
- **`viz.ts` (`PatternViz3D`)**: the main three.js scene owner — camera, renderer, `OrbitControls`, the lil-gui `Controls` panel (`show_normals`/`show_edges`/`show_nodes`/`show_forces`/`show_row_markers`), raycasting/click/hover/paint interaction, and the instanced meshes for nodes/tubes/stitches (`inst_nodemesh`/`inst_tubemesh`/`inst_knitmesh` — one `THREE.InstancedMesh` shared across all nodes for perf, indexed by node ID). `computeKnits()` builds the initial instanced-mesh state from the graph; `stepSim()` is called every simulation tick and only updates positions (not scale/color), which is why the enlarged `start_of_row` markers only show pre-simulation. When mutating per-instance transforms outside `computeKnits()`, always build a **fresh** `Matrix4` per node — reusing one mutable matrix across a loop without resetting it will compound `.scale()` calls.
- `KnitSimModule` (top of `viz.ts`) is the loaded Emscripten module, set asynchronously when `knitsim-lib.js` resolves — code that touches it must tolerate it being `null` until then.

### C++ simulation core (`wasm/`)

- `wasm/wasm.cpp`: the Embind bindings file — this is the single source of truth for what's exposed to JS as `KnitSimModule.*`. Enum value names here (`KnitNodeTypeC_KNIT`, `KnitSideC_RIGHT`, etc.) are string-matched against JS enum values in `graph.ts`, so renaming one side requires updating the other.
- `wasm/knitsim/Graph.{hpp,cpp}` + `Elements.hpp`: the C++ graph representation (`KnitGraphC`) — heuristic layout, knit-path geometry generation, normals.
- `wasm/knitsim/KnitSim.{hpp,cpp}`: the physics stepper (`KnitSim::step`/`getForce`) — spring/repulsion-style relaxation over the graph, configured via `KnitSimConfig` (expansion/flattening/repel factors).
- Build wiring: `CMakeLists.txt` (root of `knitviz-frontend/`) + `cmake/wasm.cmake` set the Emscripten link flags (`SINGLE_FILE=1` — wasm binary is base64-embedded in the `.js`, so there's no separate `.wasm` file to serve; `MODULARIZE` + `EXPORT_NAME=KnitSimLib`; `--emit-tsd` for the `.d.ts`).

### Routing / entry points

- `src/router/index.ts`: two routes — `/` (`HomeView.vue`, a confirmation gate before entering the app) and `/viz` (`EditorsView.vue`, lazy-loaded — the actual editor+preview app).
- `EditorsView.vue` is large and is the natural place to look first for cross-cutting UI behavior (tab switching, generate/preview/simulate button wiring, node selection sync between the 3D view and the Node editor).

### Known quirks worth knowing before debugging UI issues

- `src/components/ui/Btn.vue`'s non-toggleable render path (`v-else` branch) gives every instance a hardcoded `id="btnControl"` on its hidden checkbox input — duplicate IDs across the page. Don't rely on `label[for=btnControl]` or `getElementById` to find a specific button; match by visible text instead. The actual click handling is on the wrapping `<div>`'s `@click`, not the checkbox input itself.
- "Show Preview" / "Generate from X" buttons are plain buttons (not stateful toggles bound to the button itself) — the displayed label (`Show Preview` vs `Hide Preview`) is a computed property driven by `EditorsView.vue`'s own `isRunActive` state, and doing anything with the preview requires a non-empty graph in the store first.
