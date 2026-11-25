# PROCEDURAL THREE.JS GAME-READY 3D ASSET SPECIFICATION

## 1. COORDINATE SYSTEM, SCALE & PIVOT RULES
- **Coordinate Space:** Right-handed, `+Y` Up, `+Z` Forward, `+X` Left (standard glTF 2.0 / Three.js space).
- **Units:** Strict real-world metric scale (`1.0 unit = 1.0 meter`). Never output oversized or miniature assets unless the real object is miniature (e.g., a 100ml beaker must have height ~`0.075` and radius ~`0.025`).
- **Root Pivot (`0, 0, 0`):**
  - Tabletop and floor-standing assets MUST rest flush on the `Y = 0` plane (`boundingBox.min.y === 0.0`) centered on `X = 0, Z = 0`.
  - Wall-mounted assets MUST have their back mounting plane flush with `Z = 0`, centered on `X = 0, Y = 0`.
  - Handheld tools MUST have their primary grip center at `(0, 0, 0)` aligned along the `+Z` or `+Y` axis.
- **Transform Cleanliness:** Bake static rotations and scales into `BufferGeometry` (`geometry.rotateX()`, `geometry.scale()`, `geometry.translate()`). Every static mesh node MUST have `position: (0,0,0)`, `rotation: (0,0,0)`, and `scale: (1,1,1)`.

## 2. TOPOLOGY, TESSELLATION & POLYCOUNT BUDGETS
- **Triangle Budgets (Hard Limits):**
  - Small props (glassware, tools, sensors): `300 – 2,500` triangles.
  - Medium benchtop equipment (microscopes, balances, pumps): `2,500 – 8,000` triangles.
  - Large hero machines (fume hoods, MRI, spectrometers, rovers): `8,000 – 20,000` triangles.
- **Segment Caps (Prevent Over-Tessellation):**
  - Small cylindrical/lathe features (knobs, tubes, wires): `12 – 16` radial segments.
  - Primary cylindrical bodies (flasks, lenses, tanks): `24 – 32` radial segments.
  - Large hero curves (domes, gantries): max `48` radial segments.
  - Torus/TubeGeometry curves: tubular segments `<= 32`, radial segments `<= 12`.
- **Manifold & Surface Integrity:**
  - All geometries MUST be watertight and solid. Never leave open single-sided shells for glassware or housings—use `LatheGeometry` with inner and outer wall thickness (`>= 0.0015m`) or `ExtrudeGeometry`.
  - Eliminate Z-fighting: Any decal, screen, label, or overlapping trim MUST be offset by at least `0.001m` (`1mm`) along its surface normal.
  - Always call `geometry.computeVertexNormals()` after procedural vertex manipulation and ensure `side: THREE.FrontSide` is used (never rely on `THREE.DoubleSide` to hide inverted normals).

## 3. SCENE HIERARCHY & STRICT NAMING CONVENTION
Every exported scene MUST follow this exact node structure so game engines can auto-parse colliders, sockets, and animations:

- `ROOT_<snake_case_asset_name>` (`THREE.Group` at `(0,0,0)`)
  - `GEO_<material_group_name>` (`THREE.Mesh` — merged static geometry sharing one material)
  - `PART_<interactive_name>` (`THREE.Group` — movable sub-assemblies like doors, microscope turrets, knobs, lids, needles)
    - Pivot Rule: Place the `PART_` group's `position` at the exact mechanical hinge/axle center, and offset its child mesh geometry relative to that pivot so rotating `PART_.rotation` animates cleanly.
  - `SOCKET_<purpose>_<index>` (`THREE.Object3D` — empty transform nodes with zero geometry)
    - Standard socket names: `SOCKET_liquid_01` (pour/fill point), `SOCKET_sample_01` (slot for tubes/slides), `SOCKET_grip_01` (VR/hand IK target), `SOCKET_cable_01` (wire/hose port), `SOCKET_screen_01` (UI/shader anchor).
  - `COL_<box|cylinder|sphere>_<index>` (`THREE.Mesh` — simplified invisible proxy primitives for engine collision generation; keep total collision primitives `<= 8` per asset).

## 4. DRAW-CALL OPTIMIZATION (GEOMETRY MERGING)
- Static parts that do NOT animate independently and share the same material MUST be merged into a single `BufferGeometry` using `BufferGeometryUtils.mergeGeometries()` before export.
- Before merging, ensure all geometries share identical attributes (`position`, `normal`, `uv`) and are either all indexed or all non-indexed (`toNonIndexed()`).
- Target `1 – 5` total draw calls (material slots) per complete asset.

## 5. PBR MATERIALS & PROCEDURAL TEXTURES
- **Allowed Material Classes:** Use ONLY `THREE.MeshStandardMaterial` or `THREE.MeshPhysicalMaterial` (for transparent glass/liquids). Do NOT use `MeshBasicMaterial`, `MeshLambertMaterial`, or `MeshPhongMaterial`.
- **Physically Valid PBR Ranges:**
  - Dielectrics (plastic, ceramic, rubber, painted metal): `metalness: 0.0`, `roughness: 0.25 – 0.85`. Avoid pure `0.0` or `1.0` albedo extremes (clamp RGB between `0.03` and `0.90`).
  - Bare Metals (stainless steel, brass, copper, aluminum): `metalness: 0.88 – 1.0`, `roughness: 0.15 – 0.45`.
  - Laboratory Glass / Quartz: `MeshPhysicalMaterial` with `transmission: 0.92`, `ior: 1.47` (borosilicate), `roughness: 0.05`, `thickness: 0.005`, `transparent: true`.
  - Emissive Displays / LEDs: `emissive` color set with `emissiveIntensity: 1.5 – 3.0` and dark base albedo (`#05080a`).
- **Procedural Textures (Zero External Image Dependencies):**
  - Generate graduation markings, gauge dials, warning labels, LCD readouts, and micro-surface roughness via offscreen `<canvas>` (`THREE.CanvasTexture`) at `512x512` or `1024x1024` resolution.
  - Color Space Enforcement: Set `texture.colorSpace = THREE.SRGBColorSpace` for `map` and `emissiveMap`. Leave `texture.colorSpace = THREE.NoColorSpace` for `roughnessMap`, `metalnessMap`, `bumpMap`, and `normalMap`.
  - Material Naming: Name every material descriptively (`MAT_StainlessSteel`, `MAT_BorosilicateGlass`, `MAT_LabEpoxyBlack`, `MAT_ScreenEmissive`).

## 6. EXPORT & AUTOMATED PRE-FLIGHT VALIDATION
- Export using `GLTFExporter` with `{ binary: true, onlyVisible: true, embedImages: true }` to produce a self-contained `<asset_name>.glb`.
- **Mandatory Agent Self-Check Before Saving `.glb`:**
  1. `totalTriangles <= budgetMax` and `totalTriangles >= 50`.
  2. `NaN` check: Verify zero `NaN` or `Infinity` values exist in any `position`, `normal`, or `uv` `BufferAttribute`.
  3. Grounding check: `Math.abs(worldBox.min.y) < 0.0005` (for tabletop/floor assets).
  4. Dimension sanity check: `worldBox.getSize()` matches real-world physical bounds within `±15%`.
  5. Hierarchy check: Root starts with `ROOT_`, at least one `COL_` proxy exists, and all materials are `MeshStandardMaterial` or `MeshPhysicalMaterial`.
