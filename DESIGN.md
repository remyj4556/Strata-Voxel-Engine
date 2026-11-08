# Strata – Design Document

Status: **DRAFT v2 (elephant)** – all open questions from v1 answered by the author; not yet reviewed by goldfish.
Audience: someone with zero other context (goldfish reviewer, or the coder).

Conventions used below:
- **[CURRENT]** = what the code does today. **[TARGET]** = what must be true when the design is implemented. **[GAP]** = the difference.
- **Decision (author)** = an explicit choice by the project author, with the rationale recorded. Everything else in a [TARGET] section follows from those decisions.
- "Tunable" = a numeric constant the implementer chooses (listed in §13); it is below the level of architecture and does not need approval.
- If a behaviour is not in this document, it has not been decided (§12).

---

## 1. Project goal

Strata is a from-scratch C++20 / OpenGL 3.3 voxel terrain engine. It is a **performance showcase**: it demonstrates a concurrent chunk-streaming pipeline (async generation, loading and greedy meshing on a worker pool; the main thread limited to GL work) that keeps thousands of chunks resident at a stable frame rate.

Reference numbers claimed by the author (32³ chunks, single-octave heightmap, RTX 3050 + Ryzen 7 5800H): 6,000+ resident chunks, ~70 FPS, ~5.5 ms mesh / chunk, ~0.1 ms generation / chunk, ~150 vertices / chunk. New features must not regress these.

**Decision (author):** performance showcase, not (yet) a game. Rationale: priority order is (1) throughput and frame stability, (2) concurrency correctness, (3) features.

### Scope
In scope (to be built; all are **[GAP]** today):
- Race-free chunk state machine (§4.3)
- Disk persistence of edited chunks (§9)
- Transparency (§10)
- Block editing (§11)
- Day/night cycle (§8.4)
- Window resize, working MSAA, distance fog, frustum culling, upload budget (§8.3)
- Automated tests (§12)

Out of scope: gameplay beyond editing (entities, inventory, physics/collision – the camera is a free-fly camera), networking, audio, non-Windows platforms.

**Decision (author):** Windows / MSVC (Visual Studio, toolset v145, x64) is the **only** supported target. Rationale: simplifies build and path handling. README's `-pthread` note and the Win32 configurations are stale.

---

## 2. Technology stack

| Concern | Choice |
|---|---|
| Language | C++20 (MSVC, x64 only) |
| Graphics | OpenGL 3.3 core, GLAD loader |
| Windowing/input | GLFW (prebuilt `glfw3.lib` in `third_party_include`) |
| Math | GLM (vendored, `Engine/include/glm`) |
| Debug UI | Dear ImGui (vendored) |
| Noise | FastNoiseLite (vendored header) |
| JSON | nlohmann/json (vendored header) |
| Images | stb_image / stb_image_write / stb_rect_pack (vendored) |
| Exe path | whereami (vendored) |
| Tests | Google Test (new `Tests` project in the solution, §12) |
| Build | `VoxelEngine.sln`; app project `Engine/Engine.vcxproj`; output `bin/`, intermediates `bin-int/` |

No package manager. Vendored third-party code is never edited.

---

## 3. Architecture overview

```
main.cpp ─► Game::run()  (main thread: input, camera, lights, frame loop)
              │
              ├─ ResourceManager ─► BlockRegistry (blocks.json) + TextureLibrary (texture array)
              │                         └─► GPU block-definition UBO (binding 0)
              │
              ├─ World::update(StreamTarget)           [main thread, when rescan needed]
              │     ├─ streamChunkLoads   ─► pool task: loadChunk (region file ► else genChunk)
              │     ├─ streamChunkMeshing ─► pool task: meshChunk (ChunkMesher)
              │     └─ streamChunkUnloads ─► unload_queue ─► (save task if edited) ─► erase
              │
              ├─ upload_queue  (workers ─► main)  RenderRequest{coord, opaque + transparent vertices/indices}
              ├─ unload_queue  (main ─► main)     ChunkCoordinates
              │
              ├─ RegionStore (disk) – worker-thread I/O, per-region mutex
              │
              └─ Renderer  [main thread only: ALL OpenGL calls]
                    processQueuedChunkMeshes (budgeted) → cull → opaque pass → transparent pass
```

Principles (apply to every section):
1. **Single GL thread.** Workers never call OpenGL. Meshes cross threads as plain vertex/index vectors.
2. **`ChunkMap` is the sole owner of chunk data.** Everything else holds `ChunkCoordinates` and re-fetches via `tryGet`; workers additionally hold a `shared_ptr<Chunk>` for the duration of one task.
3. **Queues carry small keys or finished products**, never mutable shared state.
4. **Chunk state is the synchronization contract** between main thread and workers (§4.3).
5. **The main thread never blocks on generation, meshing or disk I/O.**
6. **Logic units are GL-free** (§12): coordinates, chunk, mesher, registry parsing, thread pool, queues, region store must compile and run without GL/GLFW headers, so they can be unit-tested.

---

## 4. World: streaming and chunk lifecycle

### 4.1 Goal
Keep exactly the chunks within `chunk_load_radius` (default 12) chunks of the camera loaded, generated (or restored from disk), meshed and drawn, with all heavy work on worker threads. The world is **unbounded in all three axes** (chunk coordinates are `int32`); terrain height range is set by the noise (≈ ±25 blocks).

### 4.2 Architecture and implementation **[CURRENT]**
- **Chunk**: `CHUNK_SIZE = 32`; 32³ array of `block_id_type` (`uint16_t`), index `x + z*32 + y*1024`; `std::atomic<ChunkState> state`. Block id 0 = air.
- **Coordinates** (`Coordinates.hpp`): `WorldCoordinates` (int32), `ChunkCoordinates` (int32, hashed by `ChunkCoordinatesHash`), `LocalCoordinates` (uint8, 0..31). World→chunk uses floor division.
- **ChunkMap**: mutex-wrapped `unordered_map<ChunkCoordinates, shared_ptr<Chunk>>` (`tryGet / insert / erase / contains / getAllCoords`).
- **StreamTarget**: `{glm::vec3 pos; int chunk_load_radius;}` built from the camera each frame.
- **Load region**: a *sphere* in chunk units, `x²+y²+z² <= r²`.
- `streamChunkLoads`: creates missing chunks (`Unloaded`), moves each `Unloaded` chunk to `QueuedToLoad`, enqueues `loadChunk` (currently = `genChunk`).
- `genChunk` (worker): 2D Perlin (frequency 0.02) at world (x,z); `height = noise*25`; blocks with world y < height = id 1, y == height = id 3, else air. Sets `Loaded`. **[CURRENT]** ids 1 and 3 are hard-coded. **[TARGET]** see §4.6.
- `streamChunkMeshing`: a `Loaded` chunk whose 6 face-neighbours all exist and are not `Unloaded`/`QueuedToLoad` → `QueuedToMesh`, `meshChunk` enqueued; `Loaded`/`Meshed` neighbours are re-queued so borders are rebuilt.
- `meshChunk` (worker): builds a `ChunkGroup`, runs `ChunkMesher::buildGreedyMesh`, sets `Meshed`, pushes a `RenderRequest` to `upload_queue`.
- `streamChunkUnloads`: any chunk beyond radius (squared chunk distance `> r²`) → `QueuedToUnload`, coordinate to `unload_queue`; `Game` drains it: `renderer.unloadChunkMesh`, then `world.unloadChunk` (erase if still `QueuedToUnload`).

**Decision (author):** the outermost shell of the load sphere is intentionally loaded but not meshed (a chunk meshes only when all 6 neighbours exist). Rationale: provides neighbour data so inner chunks cull border faces correctly. Consequence: visible radius is effectively `radius − 1` chunks.

**Decision (author):** chunks that are entirely air are still created, meshed and uploaded as empty meshes (no special case). Rationale: uniform handling; cost is small.

### 4.3 Chunk state machine **[TARGET – strictly race-free]**

**Decision (author):** strictly race-free; the proposed protocol below is approved. Rationale: the current code lets a worker overwrite `QueuedToUnload` (chunk never erased) and lets a late mesh create a GPU mesh for an unloaded chunk (ghost mesh + leak).

States: `Unloaded → QueuedToLoad → Loaded → QueuedToMesh → Meshed → Uploaded → QueuedToUnload`. Remeshing returns `Meshed`/`Uploaded` → `QueuedToMesh`. `QueuedToUnload` is reachable from any live state (not from `Unloaded`/`QueuedToUnload`).

Protocol:
1. **Only compare-exchange transitions.** Every change is `compare_exchange_strong(expected, desired)`; no unconditional `state = X`.
2. **Main thread owns all `Queued*` transitions and `QueuedToUnload`.** Workers own only completion transitions: `QueuedToLoad → Loaded`, `QueuedToMesh → Meshed`.
3. **Worker completion is conditional.** If the CAS fails (chunk moved to `QueuedToUnload` or re-dirtied) the worker discards its result and does **not** push to `upload_queue`.
4. **Upload completion.** When the main thread consumes a `RenderRequest` it first does CAS `Meshed → Uploaded`. If the chunk no longer exists or the CAS fails, the request is dropped and no GPU mesh is created.
5. **Unload.** Main thread CAS (live state) → `QueuedToUnload`, pushes to `unload_queue`. When drained: erase GPU mesh; if the chunk is `modified` (§9) write it to disk first (the chunk stays in `ChunkMap` until the write completes); then erase from `ChunkMap`.
6. **Dirty remesh.** Each chunk has an atomic `mesh_version`, incremented whenever the chunk's blocks or its neighbour set change. `meshChunk` records it at start; on completion, if it changed, the worker does not publish: it transitions back to `QueuedToMesh` and the chunk is re-enqueued.
7. `Uploaded` must actually be set (today never) and is the only state in which the chunk's mesh exists on the GPU.
8. **Dead code to remove:** `Chunk::advanceChunkState`, `ChunkLoadRequest`, `ErrorCodes`/`errorToString`, unused `uvs[4]` in the greedy mesher, `buildNaiveMesh` (§6.2). `World::blockAtWorldPos` is kept (used by block editing, §11). `chunkStateToString` must handle all states.

### 4.4 Streaming scans **[TARGET – event-driven]**
**Decision (author):** rescan only on change. Rationale: `streamChunkMeshing` and `streamChunkUnloads` currently scan the whole sphere every frame (~7,000 mutex-locked lookups at r=12), contending with workers.

`World::update` runs the three stream functions only when a `rescan_needed` flag is set. The flag (atomic) is set when: the camera enters a different chunk, the radius changes, a chunk finishes load or mesh (worker completion), a chunk is erased after unload/save, or an edit is applied. The flag is cleared at the start of a scan.

### 4.6 Terrain generator uses block names **[TARGET]**
**Decision (author):** terrain generation refers to blocks by **name**, not hard-coded id. Rationale: ids are an internal detail of `blocks.json`; the generator should survive id reshuffles.
- `BlockRegistry` exposes name→id lookup by the `display_name` field of `blocks.json` (§7.2). Names must be unique (validated, §7.3).
- At startup the generator resolves the names it needs once and caches the ids (workers read them without locking). To preserve today's terrain: subsurface fill = `"cobblestone"` (currently id 1), surface block = `"grass"` (currently id 3). A required name that is missing from the registry is **fatal at startup** (clear message).
- Name constants live with the generator, not in `blocks.json`.

---

## 5. Concurrency primitives

### 5.1 Goal
A small hand-written task pool and queues so generation, disk I/O and meshing run in parallel while the main thread only uploads and draws. (The from-scratch pool is itself a showcase feature.)

### 5.2 Implementation **[CURRENT]**
- **ThreadPool**: N `std::thread`s waiting on a condition variable over a FIFO `std::queue<std::function<void()>>`. `enqueueTask(f, args...)` wraps the call in a `std::packaged_task`, returns a `std::future`. N = `hardware_concurrency() − 2` (unsigned).
- **ThreadSafeQueue<T>**: mutex + `std::queue`; `push`, `tryPop`, `size`, `empty`. Owned by `Game`: `upload_queue` (many workers → main), `unload_queue` (main → main).
- `concurrentqueue.h` is vendored but unused.

### 5.3 Requirements **[TARGET]**
**Decision (author):** all four apply.
1. Thread count = `max(1, hardware_concurrency() − 2)` (no unsigned underflow).
2. **Safe shutdown order:** the pool is stopped and joined before `World`, `ChunkMap`, queues and the region store are destroyed.
3. **Task exceptions are logged** (not lost in discarded futures).
4. **Pending tasks are cancelled on shutdown** (destructor discards queued tasks instead of draining; in-flight tasks finish). Rationale: fast exit. Because cancellation could drop save tasks, `Game` shutdown must first flush all `modified` chunks synchronously (§9.4) and only then stop the pool.

---

## 6. Meshing

### 6.1 Goal
Convert a chunk (plus read access to its 6 face neighbours) into the smallest practical vertex/index set, quickly, off the main thread.

### 6.2 Implementation **[CURRENT]**
- `ChunkGroup`: main + left(−X) / right(+X) / bottom(−Y) / top(+Y) / front(+Z) / back(−Z). `blockAtLocalPos(x,y,z)` redirects a coordinate that overflows on **exactly one axis** to the neighbour; a missing neighbour reads as air. Invariant: only face-adjacent cells are queried (diagonal queries, e.g. for ambient occlusion, would require changing this).
- Face ids: Back=0(−Z), Front=1(+Z), Left=2(−X), Right=3(+X), Bottom=4(−Y), Top=5(+Y). The same order is used by `directions[]`, vertex `face_id`, block-definition `face_texture_id[6]`, and the shader's normal table.
- `buildGreedyMesh`: for each of 6 directions × 32 depth slices, build a 32×32 mask (cell = block id if solid and the outward neighbour is air, else 0); grow rectangles along i then j over unvisited equal cells; emit one quad (4 vertices, indices `{0,2,1,0,3,2}`) per rectangle. Positions are chunk-local integers 0..32.
- Output: `pair<vector<Vertex>, vector<GLuint>>`.

**Decision (author):** merge rule = **identical block id**. Rationale: simple, correct, materials differ per id. Different ids sharing textures do not merge – accepted.

**Decision (author):** delete `buildNaiveMesh`. Rationale: unused dead code. (Mesher tests use a reference implementation written inside the test project, §12.)

### 6.3 Target changes
- Mesher output uses `uint32_t` indices (not `GLuint`) so it is GL-free (principle 6).
- Transparency: the mesher produces two vertex/index sets per chunk – opaque and transparent (§10).
- Face culling rules, see §10.

### 6.4 Vertex format **[CURRENT]**
One `uint32_t` per vertex; no normals/UVs.

| Bits | Field | Range |
|---|---|---|
| 0–5 | X | 0–32 |
| 6–11 | Y | 0–32 |
| 12–17 | Z | 0–32 |
| 18–20 | face id | 0–5 |
| 21–31 | block id | 0–2047 |

UVs are derived in the fragment shader from `fract(world position)` (so merged quads tile without stored UVs). Per-face texture ids and material come from the block-definition UBO indexed by block id.

**[GAP]** `Vertex.cpp`, `Vertex.inl` and `lightingShader.vs` mask block id with `0x3FFF` (14 bits) though only 11 fit; must be `0x7FF`. Effective block-id limit is 255 (§7.3).

---

## 7. Content: blocks and textures

### 7.1 Goal
Data-driven blocks: adding a block means editing `Engine/json/blocks.json` and adding textures, not changing engine code.

### 7.2 Implementation **[CURRENT]**
- `ResourceManager` builds `TextureLibrary(paths.textures, 16, 16, 4)`, then `BlockRegistry::populateDefinitions(paths.json/"blocks.json", library)`.
- `TextureLibrary`: every PNG in the textures dir is one layer of a `GL_TEXTURE_2D_ARRAY` (16×16 RGBA); name→layer map keyed by file stem; NEAREST mag, NEAREST_MIPMAP_LINEAR min; layer order = directory iteration order (so ids are looked up by name, never hard-coded).
- `blocks.json` (`Engine/json/blocks.json`, copied to `bin/json/`): per block `id`, `display_name` (unique name, e.g. `"grass"`), `model` (currently always `"cube"`; **not read by the registry** – reserved/unused), six face textures (`back/front/left/right/bottom/top_texture`), `material{specular, shininess, transparent}`. Currently ids 0–4: air, cobblestone, dirt, grass (dirt sides, grass top), iron_block (`iron_block` + `iron_block_side`). Note the air entry (id 0) has `transparent: true`.
- `BlockDefinition`: `id`, `is_transparent`, `face_texture_id[6]`, material. `GPUBlockDefinition` (48 B, std140): ivec4 + ivec2 + pad + vec4 material (spec, shininess), uploaded once to UBO binding 0 (`BlockBuffer`); GLSL declares `block_defs[255]`.
- `Paths`: exe dir via whereami; `assets/{shaders,textures}` and `json` under the exe dir. Asset files are copied to `bin/` by per-file entries in the vcxproj (new files need a vcxproj entry).

### 7.3 Requirements **[TARGET]**
**Decision (author):** block ids are **dense `0..N-1` (0 = air), `N ≤ 255`**. Rationale: matches shader UBO array and index-by-id upload. The registry validates at load and **fails loudly** (clear message, startup aborts) on: gaps, duplicate ids, duplicate `display_name`s, `N > 255`, missing/unparseable `blocks.json`, missing required keys. (Today: out-of-bounds write on gaps; silent continue on missing file in Release.)

**Decision (author):** texture problems **fall back to the `default` texture**. A block referencing a texture name that doesn't exist → log a warning and use `default`. A texture file that fails to load or is not 16×16 → log, skip the file (not uploaded). If `default` itself is missing → fatal. (Today: `getTextureID` falls off the end of the function – UB.)

Other requirements:
- `TextureLibrary` owns its GL texture (non-copyable, destructor frees it). `uploadGPUBlockDefinitions` must not leak its UBO if called twice.
- GL-free registry parsing (principle 6): `populateDefinitions` takes a texture-name→id resolver rather than a `TextureLibrary&`.
- `is_transparent` must reach the mesher (§10): the registry is read-only after startup, so workers read it without locking.

**Correction to v2:** an earlier draft said the iron textures have no block. They do: block id 4 `iron_block` uses `iron_block.png` and `iron_block_side.png`. It is simply never generated by the terrain (only placeable once block editing exists). The author's "keep" answer therefore stands trivially: keep the block and textures.

---

## 8. Rendering and application shell

### 8.1 Goal
Draw all visible meshed chunks every frame with stable frame time.

### 8.2 Implementation **[CURRENT]**
- `main.cpp`: GLFW init (GL 3.3 core), window 1920×1080 × monitor content scale, GLAD, `Game`, callbacks (framebuffer-size, cursor, scroll), ImGui init, `game.run()`.
- `Game::run` frame order: delta-time & FPS → light updates → input → `world.update` → `renderer.beginFrame` → `processQueuedChunkMeshes` → `drawChunks` → drain `unload_queue` → ImGui debug panels → swap/poll.
- `Renderer`: shader programs; `chunk_meshes` map (coord → `Mesh`); per chunk sets a `model` matrix (translate by `coord*32`) and draws. `beginFrame` sets view/projection (perspective, near 0.1, far = `render_distance`), sun/player-light uniforms, binds the texture array. `processQueuedChunkMeshes` drains the **entire** `upload_queue` each frame.
- `Mesh`: VAO + VBO + EBO; one `glVertexAttribIPointer` uint attribute; move-only GL wrappers; CPU copies of vertices/indices kept.
- Shaders: vertex unpack as §6.4; fragment: Blinn-Phong sun (ambient/diffuse/specular × `time_of_day`) + optional point "player light" (added at full strength) + constant `0.2·texture` ambient; no fog, alpha, gamma or AO.
- `Camera`: yaw/pitch fly camera; WASD, Space/Shift vertical, scroll = FOV (1–100), speed fixed at 100.
- Controls: WASD, mouse look, Space/Shift, C toggles cursor, L toggles handheld light, Esc quits.
- `DebugRegistry`: named ImGui sections (Game, Renderer, World).

### 8.3 Rendering requirements **[TARGET]**
**Decision (author):** all five. Rationale: the visible/measurable shortcomings of a showcase.
1. **Resize:** viewport and projection aspect follow the **framebuffer** size (`glfwGetFramebufferSize`, not window size – they differ on high-DPI); `Game` and `Renderer` both get the new size. A zero-size (minimized) framebuffer: skip the projection update and rendering for that frame; no divide-by-zero.
2. **MSAA:** `GLFW_SAMPLES = 4` is set **before** `glfwCreateWindow` (currently set in the `Renderer` constructor, too late).
3. **Fog:** distance fog toward the clear (sky) colour, so it follows day/night. Fog is fully opaque at or before `(r − 1)·32` (the visible radius, §4.2). The far plane is set beyond the farthest loaded point: `(r + 1)·32 + chunk diagonal` at least. `render_distance` is derived from the load radius in one place, called at startup and whenever the radius changes (today: hard-coded 500 vs 12·32 = 384, and the setter is never called).
4. **Frustum culling:** `drawChunks` skips chunks whose 32³ world-space AABB is outside the view frustum (planes extracted from projection×view).
5. **Upload budget:** `processQueuedChunkMeshes` creates at most `MAX_MESH_UPLOADS_PER_FRAME` meshes (**decision: max meshes per frame**, tunable); the remainder stay queued. Upload only counts a request that passes the `Meshed → Uploaded` check (§4.3.4).

### 8.4 Day/night cycle **[TARGET]**
**Decision (author):** time advances in **real time with a fixed day length** (tunable constant). An ImGui "Time" section provides a **Pause** toggle and a **time-of-day slider**; while paused the slider sets the time; on unpause time continues **from the current point at the same pacing** as before. A **night brightness floor** is kept: the constant `0.2·texture` ambient stays so nights never go fully black.

Design: a single `time_of_day ∈ [0,1)` (day phase) lives in `LightManager`; it drives sun direction/intensity, the moon light (already in `LightManager`, currently unused by the shader – it should contribute dim light at night), and the clear/sky colour (which the fog follows). Today `game_time` is a constant 1.0, so none of this varies. The handheld player light is unchanged (not dimmed by day). Exact curves are implementation detail.

### 8.5 Shell cleanup **[GAP / TARGET]**
- Edge-triggered key state (`c_key_pressed`, `l_key_pressed`) moves from file globals into `Game`.
- Failure paths: check `glfwInit` and monitor/window creation, call `glfwTerminate` on early exit, set a GLFW error callback.
- `delta_time` from `glfwGetTime()` must not be narrowed to `float` before subtraction; the FPS average must not divide by zero.
- Debug label "Mesh Upload Queue" prints `unload_queue.size()` – fix to show `upload_queue.size()`.
- When ImGui wants keyboard/mouse (`io.WantCaptureKeyboard/Mouse`), game input is ignored (needed once the Time UI exists).
- **Decision (author):** remove the debug light cube: `light_shader`, `lightCubeShader.vs/.fs`, `Renderer::renderDebugLight` and their vcxproj entries. Rationale: it expects a `vec3 aPos` mesh that `Mesh` no longer supports (dead). (Block editing adds a new, separate outline renderer, §11.)

---

## 9. Persistence **[TARGET – not started]**

### 9.1 Goal
Player edits survive unloading and restarts, without slowing the streaming pipeline.

### 9.2 Decisions (author)
- **Region files.** Chunks are grouped into fixed-size cubic regions (tunable edge, in chunks) – one file per region – instead of one file per chunk. Rationale: thousands of chunks resident; avoids thousands of tiny files.
- **Only edited (`modified`) chunks are saved.** Rationale: untouched terrain is deterministic from the world seed + noise parameters, so saving it wastes I/O and disk. A chunk has a `modified` flag set by block edits (§11).
- **Location and safety.** Saves live next to the exe: `<exe dir>/saves/<world>/`. Every file has a **format version header**. A corrupt or unknown-version chunk record is **logged and the chunk is regenerated from noise**.
- **World seed in metadata.** Each world folder holds a metadata file with the **seed, noise parameters, and format version**; a new world picks a random seed, an existing world reuses it. `genChunk` uses the seed (`FastNoiseLite::SetSeed`) and stored parameters. Rationale: only-edited-chunks persistence requires identical regeneration.

### 9.3 Design derived from those decisions
- **Fixed-size, uncompressed slots.** A chunk record is a small header (version, chunk coords) + the raw 32³ × `uint16_t` block array (64 KiB). Records are fixed size, so a region file is a header (magic, version, presence bitmap/table) followed by slots addressed by local chunk index; re-saving overwrites the slot in place – no free-space management. (No compression was chosen; may be revisited if disk use matters.)
- **Decision (author): one world is enough.** Folder `saves/default/`, created with a fresh random seed if absent. No world picker or new-world flow is planned. Rationale: scope; the project is a performance showcase.
- **Concurrency.** All region I/O runs on pool workers (principle 5). A per-region mutex serializes access to one file. Loads: `loadChunk` first asks the region store; if a valid record exists it is used, else the chunk is generated.
- **Saving on unload.** §4.3.5: a `modified` chunk stays in `ChunkMap` in `QueuedToUnload` until its save task completes; completion (a worker-to-main signal, e.g. a queue) lets the main thread erase it and set `rescan_needed`. A chunk that re-enters the radius while saving is not reloaded until it has been erased and recreated by the next scan.
- **Known limitation:** records store numeric block ids; editing the ids in `blocks.json` invalidates existing saves (format version does not cover it).

### 9.4 Shutdown flush
Before the thread pool is stopped (§5.3), `Game` synchronously writes every `modified` chunk still in `ChunkMap`.

---

## 10. Transparency **[TARGET – not started]**

### 10.1 Goal
Blocks with `transparent: true` in `blocks.json` (e.g. glass, water) render correctly. Today `is_transparent` is parsed and ignored.

### 10.2 Decision (author): cutout + blended pass
Rationale: correct see-through blocks with bounded complexity (no per-fragment sorting).
- **Air is special.** Block id 0 is air: it is never meshed and always counts as empty space, regardless of its `transparent: true` flag in `blocks.json` (that flag on air is meaningless and ignored). "Transparent block" below means a non-air block with `transparent: true`.
- **Culling rules.** An opaque block's face is emitted if the neighbour is air **or transparent**. A transparent block's face is emitted if the neighbour is air or a *different* transparent block; faces against opaque blocks and against the *same* transparent block are culled.
- **Two meshes per chunk.** The mesher outputs an opaque set and a transparent set (separate greedy runs, same algorithm, merge rule = identical block id). `RenderRequest` carries both; the renderer keeps two GPU meshes per chunk (the transparent one may be empty).
- **Render order.** Pass 1: all opaque meshes (depth write on, no blend). Pass 2: transparent meshes with blending on, depth write off, chunks sorted **back-to-front by chunk distance** from the camera each frame. No sorting within a chunk (accepted limitation).
- **Fragment alpha** comes from the texture's alpha channel; fully transparent fragments are discarded, partial alpha blends.
- Vertex format and the block-definition UBO are unchanged: pass choice is made per mesh, not per vertex.

---

## 11. Block editing **[TARGET – not started]**

### 11.1 Goal
The player can break and place blocks; affected chunks are remeshed and saved.

### 11.2 Decisions (author)
- **Targeting:** raycast from the camera through the world with voxel traversal (DDA) up to a reach limit (tunable), using `World::blockAtWorldPos`.
- **Input:** left mouse = break (set air), right mouse = place the selected block in the empty cell adjacent to the hit face. The selected block is chosen with number keys and/or an ImGui picker (ids `1..N−1`).
- **Outline:** the targeted block is highlighted with a wireframe box. This needs a small new renderer (its own shader and a `vec3`-position line/box mesh) – distinct from the removed debug light cube (§8.5).
- **Edit safety: apply when idle.** Edits are queued on the main thread and applied only when the target chunk **and all 6 neighbours** are not `QueuedToMesh` (no meshing worker can be reading them); otherwise retried next frame. Rationale: no locks and no copies on hot paths; accepted cost is a brief edit delay.

### 11.3 Design derived from those decisions
- Applying an edit (main thread): write the block, set the chunk's `modified` flag, bump `mesh_version` (§4.3.6), CAS the chunk `Meshed/Uploaded → QueuedToMesh`; if the edited cell lies on a chunk face, also re-queue the adjacent chunk(s) the same way; set `rescan_needed`.
- Edits targeting a chunk that is not at least `Meshed` (not ready) are rejected.
- Placement/breaking is sound with respect to persistence: the chunk is `modified` and will be saved on unload or shutdown (§9).
- **Decision (author):** placing a block inside the camera's own cell is allowed. Rationale: there is no collision (free-fly camera), so it causes no problem.

---

## 12. Testing

**Decision (author):** targeted automated tests with **Google Test**, in a separate `Tests` project in the same solution. Rationale: correctness of the concurrency model and the pure logic is the project's core claim.

Scope: pure, GL-free logic (principle 6):
- Coordinate conversions (world/chunk/local, negative coordinates).
- Greedy mesher correctness against a **reference mesher written in the test project** (naive per-block face emission), comparing the covered visible surface; includes chunk borders and neighbours, transparent/opaque rules.
- State-machine/concurrency stress: the CAS protocol of §4.3 (no lost unload, no stale publish, no ghost upload) with many threads.
- ThreadPool: shutdown with queued tasks, exception logging, minimum thread count.
- Region store: write/read round-trip, corrupt record → regenerate path, version mismatch, concurrent access.
- BlockRegistry validation (gaps, duplicates, >255, missing keys, missing file).

Rendering and window behaviour are verified manually (run the app, ImGui stats). No GPU tests.

---

## 13. Tunables (implementer's choice; not architecture)
Chunk upload budget per frame; reach distance for editing; region edge length (chunks); day length (seconds); fog start/end within the §8.3 constraints; sun/moon curves; default selected block; noise parameters beyond seed; log format/destination.

## 14. Remaining open questions (small; goldfish should challenge these too)
1. **Rescan correctness.** The event-driven scan (§4.4) lists its trigger events; goldfish should look for a missing event that could leave a chunk permanently unmeshed.
2. **`model` field in `blocks.json`.** Every block has `"model": "cube"`, but the registry ignores it. Is it reserved for future non-cube blocks (then keep and document), or should it be removed?

## 15. Documentation hygiene
- README: remove `-pthread`/cross-platform wording; document the Visual Studio build, that `blocks.json` lives in `Engine/json/`; include `Uploaded` in the state description.
- `CLAUDE.md` currently says `blocks.json` is under `assets` and that all dependencies are vendored in `include/src`; GLFW is a prebuilt library in `third_party_include`. Both statements need correcting (CLAUDE.md edits are the user's call under the workflow rules, so flagged here only).
