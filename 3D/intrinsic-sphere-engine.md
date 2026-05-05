# The Intrinsic Sphere Engine

## A 3D Engine Where the Shape of Space Is the Coordinate System

---

## 1. What This Is

This document describes a 3D game engine built on a single radical premise: the world is a sphere, and the sphere is not an object placed inside a Euclidean 3D space. The sphere *is* the space. There is no outer coordinate system. There are no global X, Y, Z axes. Every position, direction, distance, and movement is described in the language of the sphere's own surface — the same way we describe things on Earth.

We say "400km northeast and 200m above sea level." We don't say "the Earth is centered at the origin and this point is at (4,201,033.7, 1,847,291.2, 4,503,118.9)." The engine adopts this philosophy absolutely. The sphere is the entire universe. Everything is expressed in planet language.

This is not a non-Euclidean novelty engine. It is not designed to make the player feel disoriented. Locally, at human scale, it feels exactly like standing on a planet. But it is not a faithful simulation of living on a planet in Euclidean space. It is something stranger: a universe whose geometry is S² × ℝ — a sphere crossed with a line. This is a space that has never existed in physical reality. It feels like Earth almost everywhere, but it has properties no real planet possesses. Light follows the surface. A horizontal beam circumnavigates the globe and returns. You can see the back of your own head. This isn't a bug or a large-scale illusion. It's a fundamental feature of the geometry, present at every scale, and no amount of increasing R can remove it.

---

## 2. The Core Idea: Intrinsic vs. Extrinsic Geometry

In every existing 3D engine, a sphere is an object *embedded* in Euclidean 3D space. Every point on it secretly has an (x, y, z) address. The sphere is a guest in Euclid's house.

This engine evicts Euclid. The sphere's surface is the only reality. There is no ambient space. No "outside." No "above the sphere in absolute terms." The mathematical framework for this is called *intrinsic Riemannian geometry* — the geometry of curved spaces described from the inside, without reference to anything external.

The practical consequence is that every common operation — movement, distance, direction, line of sight, projectile trajectories — is defined natively on the sphere. No operation requires converting to 3D Euclidean coordinates, performing the computation, and converting back. The sphere is not a second-class citizen bolted onto a flat engine. It is the engine.

The only moment Euclidean coordinates appear is at the very end of the pipeline, when geometry must be handed to the GPU for rasterization. This is a mechanical, one-way conversion — like converting Celsius to Fahrenheit for a display. The engine thinks in planet language; the GPU receives a translation.

---

## 3. Why Not Just Place a Sphere in a 3D Engine?

Because it makes every surface operation harder than it needs to be.

"Move forward 5 meters" becomes: compute tangent vector in 3D, step along it, the point is now off the sphere, project back onto the surface, recompute the up vector, recompute the orientation basis. Every frame. For every entity.

"How far is that object?" becomes: take two 3D positions, realize you can't use Euclidean distance because it goes through the sphere's interior, compute great circle distance anyway — the intrinsic math, but with extra steps.

"Spawn an enemy 200m north of the player" becomes: derive "north" from the surface normal cross product with some reference axis, which breaks at the poles.

Every one of these operations is fighting the engine's native language. You're constantly converting into Euclidean, doing the thing, then converting back. The extrinsic approach adds unnecessary complexity because it forces every surface operation through a 3D detour the game logic never wanted.

In the intrinsic approach:

- "Move 5m north" → `dφ = 5/R`. Done.
- "Distance to that object" → one great circle formula. Native.
- "Spawn 200m north" → `φ_new = φ + 200/R`. Done.
- "Is this in front of me?" → compare bearing angles. Two numbers.

The game-logic layer becomes simpler, not harder.

---

## 4. The World

A world is defined by exactly three things:

```
World:
    R               — radius in meters
    axis            — defines the north and south poles
    prime meridian   — defines θ = 0
```

**R** is the only geometric parameter. It determines the circumference (2πR), the surface area (4πR²), the horizon distance, how fast geodesics close — everything.

**The axis** is the first structural choice. It defines the poles, which defines latitude, which defines the equator and hemispheres. Without an axis, there is no latitude. The coordinate system cannot exist without it. On Earth, the axis comes from the planet's rotation. In this engine, it can be a design choice.

**The prime meridian** is the second structural choice. It anchors longitude to a specific line on the sphere. Geometrically arbitrary — moving it just shifts all θ values by a constant. But necessary for coordinates to refer to specific locations.

These three choices — a scale, an orientation, and a reference line — are the entire foundation. Nothing else is assumed. Everything else is derived.

---

## 5. The Metric Tensor

The single mathematical object that encodes the world's geometry:

```
g = | R²cos²φ    0   |
    |    0       R²  |
```

This 2×2 matrix, defined at every point on the surface, tells you how to measure distances and angles in the local tangent plane. It replaces all three global Euclidean axes. It *is* the geometry.

From this one object, everything follows: geodesics, parallel transport, area, curvature. Change this matrix and you change the world — its shape, its topology, its physics — without touching any other code.

---

## 6. Formal Definitions

### 6.1 Position

A position is a triple:

```
P = (θ, φ, h)
```

- θ is longitude, range −π to π.
- φ is latitude, range −π/2 to π/2.
- h is height above the surface in meters, ≥ 0 (or negative for tunnels).

θ and φ are coordinates *on* the world. h is the one concession to a third dimension, measured in meters, treated as Euclidean.

### 6.2 Direction

A direction exists only at a position. It has no meaning on its own. You cannot say "north" without saying where you're standing.

```
D = (β, ψ)    defined at some P
```

- β is bearing: 0 = north, π/2 = east, π = south, 3π/2 = west. Measured clockwise from north, like a compass.
- ψ is pitch: 0 = horizontal, +π/2 = straight up, −π/2 = straight down.

A direction is a unit vector in the tangent space at P. Two directions at different positions cannot be directly compared. To compare them, one must be parallel transported to the other's location.

### 6.3 Displacement

The "earth way" of expressing how to get from A to B. Not a single number. A tuple:

```
Δ = (d, β, Δh)
```

- d is the great circle arc length along the surface, in meters.
- β is the initial bearing from A toward B.
- Δh is the height difference: h_B − h_A, in meters.

Expressed naturally: "the target is 3km to the northeast and 200m up."

The formulas:

```
d = R · arccos(sinφ_A sinφ_B + cosφ_A cosφ_B cos(θ_B − θ_A))

β = atan2(sin(θ_B−θ_A) cosφ_B,  cosφ_A sinφ_B − sinφ_A cosφ_B cos(θ_B−θ_A))

Δh = h_B − h_A
```

### 6.4 Velocity

A velocity is a direction plus a speed, at a position:

```
V = (v_s, β, v_h)    at some P
```

- v_s is surface speed in m/s.
- β is the bearing of movement.
- v_h is vertical speed in m/s.

### 6.5 Orientation

A rigid object has a full 3D orientation in its local tangent space:

```
O = (β, ψ, ρ)    at some P
```

- β is yaw — compass bearing the object faces.
- ψ is pitch — forward/back tilt.
- ρ is roll — left/right lean.

These are local tangent-space rotations. They only make sense at a specific (θ, φ). Move the object, and you must parallel transport the orientation.

---

## 7. Movement

### 7.1 Position Update

Each frame, given surface speed v_s, bearing β, vertical speed v_h, and time step dt:

```
dφ = (v_s · cosβ) / R · dt
dθ = (v_s · sinβ) / (R · cosφ) · dt
dh = v_h · dt
```

The `1/cosφ` on θ is the metric doing its job. Longitude steps grow near the poles because meridians converge.

### 7.2 Parallel Transport

As an object moves, its bearing drifts even without turning. This is the curvature of the world expressing itself:

```
dβ = sinφ · dθ
```

This single line is the entire Christoffel symbol machinery for a sphere collapsed into one correction. Skip it and straight-line walking curves incorrectly. Include it and great circles emerge naturally.

For full orientation transport:

```
β → β + sinφ · dθ
ψ → ψ    (unchanged)
ρ → ρ    (unchanged)
```

Only yaw drifts. Pitch and roll live in the local Euclidean regime and don't feel the sphere.

### 7.3 Geodesic (Straight Line on the Surface)

An object moving on the surface at constant speed with no turning input. No special code — just integrate:

```
each frame:
    dφ = (v · cosβ) / R · dt
    dθ = (v · sinβ) / (R · cosφ) · dt
    dβ = sinφ · dθ
```

The object traces a great circle. After distance πR, it reaches the antipodal point. After 2πR, it returns to the start. No wrapping logic. No boundary detection. The math does it alone.

---

## 8. The 3D Metric: S² × ℝ (Sphere Cross a Line)

The world's geometry for the full three-dimensional space (surface + height) is the product metric:

```
ds² = dh² + R²(cos²φ dθ² + dφ²)
```

Critically, there is no (R+h)² term. Height does not modify the surface geometry. This is a deliberate and fundamental choice. It means:

- Two people at different altitudes agree on surface distances.
- Vertical poles planted on the surface remain parallel at every height (they do not diverge).
- The surface and the height dimension are fully independent.

This differs from Euclidean 3D space written in spherical coordinates, which would have (R+h)². That formulation represents an extrinsic embedding. The product metric represents a genuinely different space — one where "up" is independent of the surface, which is exactly the engine's philosophy.

At local scales, the difference between the two is unmeasurably small. At planetary scale, the product metric preserves the clean separation between globe-layer and local-layer math.

### 8.1 The Circumference Is the Same at Every Height

This is the single most important and counterintuitive property of S² × ℝ. At h = 0, one full trip around the world is 2πR. At h = 10km, one full trip is still 2πR. At h = a million kilometers, still 2πR. Height doesn't stretch the sphere.

In real Euclidean 3D, a point at height h above a sphere is on a sphere of radius R+h, with circumference 2π(R+h). Going higher gives you more room. Planes at cruising altitude cover more ground per orbit. In S² × ℝ, this does not happen. There is no "bigger sphere" at higher altitude. The sphere and the height axis don't interact.

Consequences:

A plane at 10km altitude and a person on the ground, both moving at the same speed on the same bearing, stay directly above/below each other forever. The plane doesn't "pull ahead." At 10km on an Earth-sized sphere, the error is about 0.16% — invisible in gameplay. On a small sphere (R = 500m), it's more noticeable but still consistent with the world's own rules.

A "straight stick" tilted at an angle — its surface footprint shrinks as you tilt it upward. This is because the high end of the stick doesn't get "more sphere" to spread across. In the extrinsic embedding, a tilted stick is an Archimedean spiral: `r = R + α·R·tan(ψ)` in polar coordinates, where ψ is the tilt angle from horizontal. From inside the world, it looks like a perfectly straight stick at an angle.

### 8.2 How S² × ℝ Differs from Reality

This world is NOT a simulation of a planet in Euclidean space. It is a genuinely different universe:

- On a real planet, a horizontal light ray escapes tangentially into space. In S² × ℝ, it follows the surface and wraps around, because there is no "escape" — the horizontal geodesic stays horizontal forever.
- On a real planet, vertical poles diverge as they go up (radial lines fan out). In S² × ℝ, they stay parallel.
- On a real planet, orbiting at higher altitude covers more ground per lap. In S² × ℝ, it doesn't.

These differences are all consequences of the missing (R+h)² term. They are unmeasurable at local scales (building, city). They are noticeable at planetary scales. They are not bugs — they are the physics of this universe.

---

## 9. Light, Rays, and the Geodesic in 3D

Light travels in straight lines. In this world, "straight" means a geodesic of the full 3D metric. In the product metric S² × ℝ, a geodesic decomposes cleanly into independent components:

- The surface component follows a great circle.
- The height component moves at constant velocity (linearly).

A ray fired with direction (β, ψ) at speed c:

```
Surface speed: v_s = c · cosψ
Vertical speed: v_h = c · sinψ

Horizontal path:
    dφ = v_s · cosβ / R · dt
    dθ = v_s · sinβ / (R·cosφ) · dt
    dβ = sinφ · dθ

Vertical path:
    dh = v_h · dt
```

This decomposition is not an approximation. It is a theorem: geodesics in a product manifold decompose into geodesics in each factor.

### 9.1 Consequences by Pitch

**ψ = 0 (horizontal):** All speed goes to the surface. The ray follows a great circle at constant height forever. It circumnavigates the sphere and returns. On a featureless sphere with no atmosphere, you would see the back of your own head.

**ψ = −90° (straight down):** All speed goes vertical. No surface movement. Hits the ground directly below.

**ψ = −5° (slightly downward):** Almost all speed goes to the surface. The ray follows a great circle while slowly descending. Hits the ground far away.

**ψ = −18.8° (looking at a rock 5m away, 1.7m below):** The surface component traces a 5m arc. The vertical component descends 1.7m. Over 5m, a great circle arc is indistinguishable from a Euclidean straight line. The result is identical to standard rendering.

### 9.2 What You See

Every ray from the camera has a direction (β, ψ).

- Rays with ψ < 0 hit the ground at some distance. This is normal vision of the terrain.
- Rays with ψ > 0 go upward. They never hit the ground. They see sky.
- Rays with ψ = 0 stay at camera height, follow the surface, and wrap the globe.

The horizon emerges naturally. As ψ approaches 0 from below, the ground-hit distance increases until it exceeds practical render range. There is no explicit horizon line in the code. It's a geometric consequence.

On a perfectly smooth, atmosphereless sphere, a perfectly horizontal ray returns to the viewer. The entire world is visible in the horizontal plane, compressed into a ring at the exact vertical center of the field of view. Everything above that ring is sky. Everything below is ground. The ring itself is the entire planet, seen edge-on.

In any realistic scenario — terrain, atmosphere, draw distance limits — this effect would be obscured. The world would look like Earth locally. But the effect is real and fundamental. It is not a rendering artifact or a large-scale illusion. It is a consequence of the product metric S² × ℝ, which differs from real Euclidean 3D space. On a real planet, horizontal light escapes tangentially into space. In this world, it follows the surface. This distinction cannot be removed by increasing R. It is a property of the universe, not a scale-dependent illusion. (See Section 17, "Is this world the same as living on a real planet?")

---

## 10. Derived Operations

### 10.1 Horizon Distance

How far the player can see before the ground curves away:

```
d_horizon = R · arccos(R / (R + h))
```

For h = 1.7m and R = 6,371,000m: approximately 4.7km.

### 10.2 Bearing From A to B ("Look At")

```
β = atan2(sin(θ_B−θ_A) cosφ_B,  cosφ_A sinφ_B − sinφ_A cosφ_B cos(θ_B−θ_A))
```

Returns a bearing in the player's local tangent plane. No 3D vectors.

### 10.3 Visibility / Line of Sight

Can player at P_A see object at P_B?

```
d_horizon_A = R · arccos(R / (R + h_A))
d_horizon_B = R · arccos(R / (R + h_B))
d_AB = great circle arc distance

visible if d_AB < d_horizon_A + d_horizon_B
```

Two tall objects see each other over a longer distance because both peek over the curvature.

### 10.4 Area

A surface patch between (θ₁, φ₁) and (θ₂, φ₂):

```
A = R² · |θ₂ − θ₁| · |sinφ₂ − sinφ₁|
```

If distributing objects uniformly, weight by cosφ to avoid clustering at the poles.

### 10.5 Collision

Two objects at P_A and P_B with radii r_A and r_B:

```
d = R · arccos(sinφ_A sinφ_B + cosφ_A cosφ_B cos(θ_B − θ_A))
colliding if d < r_A + r_B AND |h_A − h_B| < vertical threshold
```

Great circle distance check plus flat height check. The two regimes stay decoupled.

### 10.6 Terrain

Terrain height is a function defined on the sphere:

```
h_terrain(θ, φ) → meters above the reference sphere
```

Stored as an equirectangular heightmap, cube map, procedural noise, or any other format. The engine only needs to evaluate h at any (θ, φ). An object's altitude above ground is:

```
h_above_ground = h − h_terrain(θ, φ)
```

---

## 11. Rendering

Every frame, from the player at position P_player facing bearing β_player:

For each object at (θ_i, φ_i, h_i):

```
1.  d = great circle distance from player to object
2.  if d > d_horizon (accounting for both heights): cull
3.  β_obj = bearing from player to object
4.  β_rel = β_obj − β_player

    x_local = d · sin(β_rel)
    z_local = d · cos(β_rel)
    y_local = h_i − h_player

5.  Apply standard perspective projection
```

The player is always at the origin of the render space. The world is re-expressed around them every frame. No seams, no patches, no boundaries. The sphere is invisible to the renderer. It determined where everything got placed, then vanished.

For the final GPU submission, the camera position and view matrix come directly from the SO(3) frame matrix F (see Section 14):

```
cam_pos = R · F.up              (the up column is the surface normal = position/R)
forward = F.forward
up      = F.up
right   = F.right
```

The view matrix is the frame matrix itself — no trigonometric conversion needed. The GPU never knows it's on a sphere.

---

## 12. Local Euclidean Frames

At the scale of a building, the curvature is negligible. A flat base placed on the ground "clips" into the curved surface by an amount called the sagitta:

```
sag ≈ d² / (2R)
```

For a 30m building on an Earth-sized sphere: 0.07mm. For 100m: 0.8mm. For 1km: 8cm.

The engine provides local Euclidean frames for interior and architectural work:

```
LocalFrame:
    anchor: (θ, φ, h)
    facing: β
```

Within a local frame, developers use plain (x, y, z) offsets in meters. The engine converts silently:

```
θ_object = θ_anchor + (x · sinβ + z · cosβ) / (R · cosφ)
φ_object = φ_anchor + (x · cosβ − z · sinβ) / R
h_object = h_anchor + y
```

This is not a betrayal of the intrinsic philosophy. It is exactly how Earth works. We use latitude and longitude for navigation and plain meters for room dimensions. Nobody describes their living room in spherical coordinates. Locally, the planet speaks Euclidean. The engine should too.

The crossover scale where curvature begins to matter:

```
d ≈ √(2R · ε)
```

For ε = 1cm on an Earth-sized sphere: about 350m. Below this, meshes can be flat. Above this, they should conform to the surface curvature.

---

## 13. Mesh and Object Import

When importing a mesh (a character, a building, a prop) into the world, there is a design decision: what does the mesh's geometry mean in this world?

### 13.1 Two Import Modes

**Rigid (default for small objects).** The mesh is placed in the local tangent frame at its anchor point. Its vertices are Euclidean offsets in meters. A 1m cube is a 1m cube. No subdivision, no conforming. Identical to how any engine imports a mesh. The curvature error (the sagitta) is negligible at small scales.

**Conformed (for large objects).** The mesh is "draped" onto the surface. Edges longer than a threshold are subdivided, and intermediate vertices are projected onto the sphere at the correct (θ, φ, h). The mesh follows the curvature. A 10km road follows the surface instead of hovering above it at its midpoint.

For small objects (under a few hundred meters on an Earth-sized sphere), the two modes produce identical results. The developer picks whichever makes sense.

### 13.2 Intrinsic-Correct vs. Extrinsic-Correct

A conformed object has a subtle rendering question. An entity's width is measured as arc distance (Δθ · R). In the intrinsic view, this looks the same at every height — because the product metric says horizontal distance doesn't change with h. But in a Euclidean embedding (the extrinsic/god view), a fixed angular span maps to a wider Euclidean width at higher h:

```
Euclidean width at height h = width · (R + h) / R
```

So a square entity (equal width and height in intrinsic units) looks like a perfect square in the intrinsic view but a trapezoid in the extrinsic view — wider at the top. This is not distortion. It's the correct embedding of an intrinsically square object.

**Intrinsic-correct (default):** The object looks right from inside the world. The extrinsic view shows the embedding distortion. This is correct for game objects — things the player interacts with should look right from inside.

**Extrinsic-correct:** The object has constant Euclidean dimensions. It looks right in the extrinsic view. The intrinsic view shows it as a trapezoid (narrower at top). Used for demonstration or for objects "aware of" the embedding.

The distinction only matters when objects are large relative to R. For a 20px entity on R = 300: ~7% top-to-bottom width difference (noticeable). For a 20px entity on R = 3000: ~0.7% (invisible).

### 13.3 Rotation of Conformed Objects

Rotating a rigid object is trivial — it happens in the local tangent frame. But rotating a conformed object requires re-conforming: rotate the source geometry's intent, then re-drape onto the surface. The shape of a geodesic path depends on which direction it runs, so changing orientation changes the shape.

The engine stores the source mesh plus the current orientation, and regenerates the conformed mesh when orientation changes. This is not expensive — subdivision and projection are simple operations — but it's not a single matrix multiply.

---

## 14. Internal State Representation: SO(3)

The formal definitions in Section 6 describe the developer-facing language: positions as (θ, φ, h), directions as (β, ψ), movement as geodesic integration with parallel transport. This is the planet language. Developers think in it. The API speaks it.

But (θ, φ) has a problem: the poles. At φ = ±π/2, cos(φ) = 0, and the longitude update `dθ = v·sinβ / (R·cosφ)` divides by zero. Walking through the north pole — a perfectly smooth, unremarkable motion from the player's perspective — causes four simultaneous discontinuities in the coordinates: φ reverses direction, θ jumps by π, β jumps by π, and the equation of motion becomes singular. The player did nothing. The coordinates panicked.

This is not a bug. It's a topological fact: a sphere cannot be covered by a single coordinate chart without at least one singularity. The (θ, φ) system concentrates this singularity at two points.

The solution is the same pattern Unity uses for rotations. Unity stores orientations as quaternions internally — singularity-free — and exposes Euler angles to developers. This engine stores entity state as SO(3) matrices internally — singularity-free — and exposes (θ, φ, β) to developers.

### 14.1 The Frame Matrix

A player's full state (position + orientation on the surface) is a 3×3 orthogonal rotation matrix:

```
F = | right_x    forward_x    up_x |
    | right_y    forward_y    up_y |
    | right_z    forward_z    up_z |
```

Three column vectors, each a unit vector in ℝ³:

- **up** = the surface normal at the player's location. Also encodes the position: the player is at the point R · up on the sphere.
- **forward** = the direction the player faces, tangent to the surface.
- **right** = cross product of forward and up. Completes the orthonormal basis.

This single matrix encodes position, heading, and orientation simultaneously. No θ, no φ, no β. No singularity.

### 14.2 Movement as Rotation

Walking forward at speed v for time step dt is a rotation of the entire frame around the **right** axis by angle v·dt/R:

```
F → R_right(v·dt/R) · F
```

Turning by angle dα is a rotation around the **up** axis:

```
F → R_up(dα) · F
```

Strafing is a rotation around the **forward** axis (or more precisely, a rotation around **up** followed by forward movement — but it reduces to a rotation around **forward** for the position component).

Every possible movement is a rotation. The composition of rotations is a rotation. There is no special case at the pole. There is no special case anywhere. The north pole is just another rotation matrix, and multiplying by it works identically to multiplying by any other.

### 14.3 Parallel Transport Is Free

In the (θ, φ, β) representation, parallel transport requires a manual correction: `dβ = sinφ · dθ`. That correction compensates for a coordinate artifact — the fact that the "north" reference direction changes from point to point in a way the coordinates don't track.

In the matrix representation, there is nothing to correct. When the frame rotates forward, the forward and right vectors rotate with it, maintaining their relationship to the surface automatically. Parallel transport is built into matrix multiplication. The correction term was never real physics — it was patching a deficiency in the coordinate encoding.

### 14.4 Extracting Coordinates

When the engine needs (θ, φ, β) — for terrain lookup, spatial indexing, serialization, networking, or the developer API — it extracts them from the matrix:

```
φ = arcsin(up_y)
θ = atan2(up_z, up_x)

east  = (-sinθ, 0, cosθ)
north = (-sinφ cosθ, cosφ, -sinφ sinθ)
β = atan2(dot(forward, east), dot(forward, north))
```

The coordinates are a *read-only view* into the state. They are derived on demand. They never drive the simulation. The matrix is the truth; the coordinates are a convenience.

### 14.5 The Full Internal State

```
Entity:
    F: 3×3 rotation matrix    (position + orientation on the surface)
    h: scalar                  (height above surface, in meters)
```

One matrix and one number. No angles anywhere in the simulation core. No singularity anywhere on the sphere. Smooth motion through both poles without a single special case.

### 14.6 On the Use of ℝ³

The matrix uses three-component vectors. Those vectors live in ℝ³. This appears to violate the intrinsic philosophy — Euclid was evicted, and here he is in the representation.

But the matrix is constrained. It's an orthogonal rotation matrix with determinant 1. It has three degrees of freedom, not nine. Those three degrees of freedom correspond exactly to (θ, φ, β). The ℝ³ components are a convenient encoding of an intrinsically three-dimensional object. The extra dimensions aren't "real" — they're bookkeeping that eliminates the coordinate singularity.

This is the same trade-off quaternions make: four components to describe three degrees of freedom. The extra component buys singularity-free interpolation. Here, nine components buy singularity-free movement. The philosophy remains intrinsic. The implementation uses the most practical tool available.

---

## 15. How Developers Talk in This Engine

### Placing things

Traditional: "Place the tree at (145.2, 0, -89.7)."

This engine: "Place the tree at 23.4°N, 47.1°E, on the ground."

Two coordinates and a height policy. "On the ground" means h = h_terrain(θ, φ). Height is always relative — to the surface, to terrain, to sea level — never an absolute Y coordinate.

### Distances

Never "the distance is 450." Always:

"The tower is 3.2km northwest and 80m up from the player."

A bearing, a surface distance, a separate height component. If someone asks "how far?" the answer is a tuple.

### Movement

Never "move by vector (1, 0, -1)." Always:

"Walk north at 5 m/s."
"Fly bearing 220 at 60 m/s, climbing at 10 m/s."

Every movement is a bearing and a speed, plus optionally a vertical speed. There is no X axis to move along.

### Relative spawning

Traditional: "Spawn at player position plus (100, 0, 50)."

This engine: "Spawn 112m from the player at bearing 63°, on the ground."

Relative placement is polar — distance and direction from a reference point. The engine computes the new (θ, φ) by integrating the geodesic.

### Regions and zones

Traditional: "The safe zone is a box from (0,0,0) to (500, 100, 500)."

This engine: "The safe zone is everything within 2km of 15°N, 30°E, up to 100m altitude."

Zones are surface patches — circles, wedges, latitude bands. Never boxes. Boxes don't exist on a sphere.

### Buildings

"The building is anchored at 41.2°N, 12.8°E, facing bearing 310°. It's 30m × 20m × 12m."

The anchor places it on the sphere. The facing orients it. The dimensions are local Euclidean. Nobody inside will notice the curvature.

### Camera

Traditional: "Camera at (50, 10, 200), look at (0, 0, 0)."

This engine: "Camera at 5.7°S, 88.3°W, 1.7m above ground, facing bearing 45°, pitch −10°."

### World size

Traditional: "The world is 4096 × 4096 units."

This engine: "The world has radius 6,371km."

One number. Circumference, surface area, horizon distance — all consequences of R.

---

## 16. Three Lines of Sphere

Conceptually, the engine's awareness of being on a sphere reduces to three things:

1. **Meridian convergence** — steps in longitude cover more arc near the poles.
2. **Parallel transport** — straight paths drift in bearing due to curvature.
3. **Great circle distance** — distance is arc length, not chord length.

In the (θ, φ, β) coordinate representation, these appear as the `1/cosφ` correction, the `sinφ · dθ` correction, and the great circle formula. In the SO(3) internal representation, the first two are absorbed into matrix rotation — they happen automatically, with no explicit correction terms. The great circle distance formula remains for spatial queries.

Remove these three things — conceptually — and you have a flat-world engine. Add them and the world closes on itself. Everything else — rendering, collision, visibility, local frames — is standard.

---

## 17. Frequently Asked Questions

### Q: A bullet fired straight ahead — what happens?

It follows a great circle along the surface at constant height. After traveling a distance of 2πR, it returns to the starting point and hits the shooter from behind. No wrapping code, no boundary detection. The geodesic equation is integrated forward and the return is a mathematical consequence.

No gravity is involved. The bullet was fired at pitch = 0, which means no vertical component. It stays at constant height because it has no reason to change height. "Forward" means along the surface. The surface is all there is.

### Q: Can you see the back of your own head?

On a perfectly smooth sphere with no atmosphere and no terrain: yes. The horizontal ray (pitch = 0) follows the surface and returns. In the camera's field of view, this appears at the exact horizontal center line. Above it is sky; below it is ground. The entire planet is compressed into a vanishingly thin ring at ψ = 0.

In a realistic scenario — terrain, atmosphere, draw distance limits — this would be obscured. But the effect is not a scale-dependent illusion. It is a fundamental property of the S² × ℝ geometry. On a real planet in Euclidean space, a horizontal ray escapes tangentially and never returns. In this world, it wraps. This is the single most striking difference between this engine's universe and physical reality. It cannot be removed by making R larger. It is a feature of the geometry itself.

### Q: Is the horizon flat or curved?

Both, and it emerges automatically. The horizon is the set of all points at arc distance d_horizon from the player, at every bearing. Whether it appears flat or curved depends on the ratio h/R.

- At h = 1.7m on an Earth-sized sphere: the horizon appears perfectly flat.
- At h = 10km (airplane): slight curvature visible.
- At h = 400km (orbit): obviously curved.

No rendering code controls this. The geometry produces it.

### Q: Does light travel in straight lines?

Yes. A straight line in this world is a geodesic of the product metric S² × ℝ. It decomposes into a great circle on the surface (horizontal component) and linear motion in height (vertical component). Over short distances, this is indistinguishable from a Euclidean straight line. Over planetary distances, the surface component wraps around the sphere.

### Q: What about "up" and "down"?

"Up" is the direction perpendicular to the surface at your location. It's well-defined everywhere but different at every point — two players on opposite sides of the world have opposite "ups." There is no global up direction.

Height is treated as Euclidean because at any reasonable scale relative to R, the curvature in the vertical direction is negligible. A 1m arc and a 1m straight line differ by approximately d³/(24R²), which for d = 1m on an Earth-sized sphere is about 10⁻¹⁵ meters — smaller than a proton.

### Q: Are objects placed on the surface slightly distorted because the ground is curved?

Technically yes. A flat-bottomed building clips into the curved surface. But the sag for a 30m building on an Earth-sized sphere is 0.07mm. The distortion is smaller than floating-point precision. In practice, objects are placed in the local tangent frame as normal meshes. The engine does not curve them. The "error" is unmeasurable.

For very large structures (above ~350m), meshes should conform to the surface curvature. "Flat" at that scale means "following the world," not "geometrically planar."

### Q: Do developers have to think in spherical coordinates all the time?

No. At local scales — interiors, buildings, vehicles — developers work in a local Euclidean frame with (x, y, z) offsets in meters. The engine converts to (θ, φ, h) silently. At navigation scales — map, quests, spawning, pathfinding — developers use (latitude, longitude, bearing, distance). The two languages are not in conflict; they're the same language at different zoom levels.

### Q: What happens at the poles?

In the (θ, φ) coordinate system, cos(φ) = 0 at the poles, making the longitude update divide by zero. This is not a bug — it's a topological fact. A sphere cannot be covered by a single coordinate chart without at least one singularity.

The solution: the simulation core doesn't use (θ, φ). It uses SO(3) rotation matrices (see Section 14). Movement is matrix rotation, which has no special case at the poles or anywhere else. The (θ, φ, β) coordinates are extracted from the matrix only when needed for terrain lookup, serialization, or the developer API. A player walking straight through the north pole experiences smooth, continuous motion. The matrix handles it without any special-case code.

### Q: How does this differ from existing non-Euclidean game engines?

Existing engines (HyperRogue, Hyperbolica, RogueViz) treat non-Euclidean geometry as novelty — the experience is "look how weird this is." They're designed to disorient.

This engine is the opposite. The curvature is gentle and large-scale. The local experience is "this feels like standing on a planet." The non-Euclidean geometry is correct but imperceptible at human scale for most interactions.

The goal is not strangeness. The goal is a world described in planet language. But see the next question — this world is not identical to a real planet.

### Q: Is this world the same as living on a real planet?

No. This is a critical distinction.

A real planet (Earth) exists in Euclidean 3D space. A horizontal light ray goes tangent to the surface and flies off into space. The ground curves away beneath it. Ships disappear hull-first. You cannot see the back of your own head, no matter how far the visibility. Light escapes the surface because the ambient Euclidean space gives it somewhere to go.

This engine's world has geometry S² × ℝ — a sphere crossed with a line. There is no ambient Euclidean space. A horizontal light ray follows the surface, circumnavigates the globe, and returns. You *can* see the back of your own head. This is not a large-scale illusion or a rendering trick. It is a fundamental property of the geometry, present at every scale. Making R larger does not remove it.

The difference comes from the metric. Real 3D space in spherical coordinates has ds² = dh² + (R+h)²(cos²φ dθ² + dφ²) — the (R+h)² means height changes the surface geometry, and horizontal rays at height h follow a sphere of radius R+h, diverging from the ground. The product metric ds² = dh² + R²(cos²φ dθ² + dφ²) has no (R+h)² — height is independent of the surface, and horizontal rays stay at constant height on a surface that never falls away.

This makes the engine's world a genuinely novel universe. It feels like Earth in almost every local interaction. But it has a property no physical planet possesses: light wraps. This isn't a bug in the simulation. It's a feature of the universe the engine creates.

### Q: What is the minimum viable prototype?

A sphere defined by R. A player state stored as an SO(3) rotation matrix plus height — exposed to the developer as (θ, φ, α). Movement as matrix rotation. A spherical-to-Cartesian conversion for rendering. A ground grid rendered as a patch around the player. Standard perspective projection.

Walk around. Drop markers. Keep walking. Find your own markers. You walked in a straight line and you're home.

### Q: Can't you just use modular arithmetic like 2D scrollers do?

For a circle (S¹), yes — modular arithmetic IS the geometry. S¹ is flat (zero intrinsic curvature). `x = x mod circumference` is the complete, exact, mathematically perfect description. The 2D prototype (see `2D/`) is equivalent to modular wrap.

For a sphere (S²), no. Modular arithmetic captures *topology* (the world loops) but not *geometry* (the world is curved). On a sphere, a step in longitude covers different arc distance at different latitudes — the cos(φ) factor. Your bearing drifts when you walk straight — parallel transport. "Straight" paths look curved in coordinate space — great circles. None of these can be expressed as modular wrap on two independent coordinates.

Wrapping θ and φ independently gives you a torus (Pac-Man topology), not a sphere. Even with corrected boundary conditions (reflection at poles), the interior geometry is wrong — mod arithmetic treats every step as the same size everywhere, but on a sphere steps in θ shrink near the poles. The metric tensor encodes this variation at every point, continuously. No boundary rule can replicate it.

### Q: Can voxels work in this world?

Yes, but they aren't Euclidean cubes. An intrinsic "cube" is a region with equal dimensions in all directions as measured by the metric: 1m of arc length east-west, 1m north-south, 1m of height. From inside the world, it looks and behaves like a perfect cube. In the Euclidean embedding, it's a wedge — sides follow great circle arcs, top face is wider than bottom face.

These intrinsic cubes tile the sphere naturally. At each latitude, the number of blocks in a ring is `floor(2πR·cos(φ) / blockWidth)`. More blocks at the equator, fewer near the poles. Each block is the same intrinsic size. No projection distortion. The only awkwardness is at the poles, where the last few rings have very few blocks.

This is fundamentally different from existing approaches (Minecraft sphere mods, Blocky Planet, Seed of Andromeda) which project Euclidean cubes onto a sphere and fight the resulting distortion. Those approaches start flat and force-bend. This approach starts curved and lets the curvature define what "cube" means.

### Q: How does this relate to general relativity?

The mathematical framework is identical — not metaphorically, structurally. The engine defines a metric tensor. Objects follow geodesics of that metric. "Straight" means "zero acceleration in the geometry's own terms." The coordinate singularity at the poles is analogous to the singularity at a black hole's event horizon in Schwarzschild coordinates — both are coordinate artifacts, fixed by switching representations (SO(3) for us, Kruskal coordinates for GR). Light and matter following the same geodesics is the equivalence principle.

This engine is 2+1 dimensional general relativity on a specific manifold (S² × ℝ with a fixed metric). GR is the same machinery on a 3+1 dimensional Lorentzian manifold with a dynamic metric. The tools — metric tensor, geodesic equation, parallel transport, Christoffel symbols, coordinate singularities — are the same. Building this engine is a hands-on introduction to the mathematical language of GR.

---

## 18. Comparison to Existing Approaches

Several projects have attempted spherical game worlds. All use the extrinsic approach: start with flat Euclidean geometry and project it onto a sphere.

**Cube-face projection (Blocky Planet, Seed of Andromeda, Planetcraft).** Divide the sphere into six faces of a cube. Each face has a flat grid. Project outward onto the sphere. Pre-distort grids to minimize stretching. Manage 12 seam edges and 8 corner vertices where faces meet. Subdivide blocks at higher altitudes to prevent stretching.

**Visual faking (World Curvature Shader).** Keep the world flat. Bend the rendered output in a shader to simulate visual curvature. No actual topology change — you can't circumnavigate.

**Teleportation wrapping (Spherical World mod).** Flat world with teleportation at boundaries. Torus topology, not sphere.

**Why these fight the geometry:** All start with Euclidean blocks and force them onto a curved surface. The distortion is mathematically guaranteed (Gauss's Theorema Egregium) and can only be minimized, never eliminated. Every project spends most of its effort on distortion management, seam handling, and altitude subdivision.

**How this engine differs:** It starts with the sphere as the coordinate system. There is no flat grid to project. No distortion to manage. No seams. The metric tensor handles the non-uniformity continuously at every point. The cost is abandoning Euclidean assumptions — but for any engine using continuous geometry (most 3D engines), those assumptions were unnecessary overhead.

---

## 19. Potential Beyond a Planet Game

The sphere is the first test case. The architecture generalizes.

**Variable curvature.** Make R a function of position. Flat plains (R huge), tightly curved hills (R small). Not height variation — actual geometric curvature variation. A region where parallel lines converge faster, where the horizon is closer, where triangles have more angle excess. Nobody has experienced this.

**Alternative topologies.** Change the boundary conditions instead of the metric. The torus (φ wraps instead of reflecting), the Klein bottle, the real projective plane. Same engine, different wrapping rules, radically different worlds.

**General relativity visualization.** Place a mass that warps the metric locally. Geodesics curve around it. Projectiles bend. Light bends. You experience gravitational lensing from a first-person perspective. This is 2+1 dimensional GR, made interactive.

**Differential geometry teaching tool.** Parallel transport, holonomy, geodesic deviation, the Gauss-Bonnet theorem — all experienced viscerally instead of proved abstractly.

**Generalized metric engine.** The deepest potential. The metric tensor becomes a configurable input. Any 2D Riemannian manifold becomes a playable world. The engine doesn't hardcode the sphere — it reads the metric and derives everything else. The shape of space becomes content, not code.

---

## 20. Architecture Summary

```
┌─────────────────────────────────────────────┐
│              DEVELOPER API                   │
│     Positions: (θ, φ, h)                    │
│     Directions: (β, ψ) at a position        │
│     Distances: (arc, bearing, Δh)           │
│     Everything in planet language            │
├─────────────────────────────────────────────┤
│          SIMULATION CORE (SO(3))             │
│     Entity state: 3×3 rotation matrix + h   │
│     Movement: matrix rotation               │
│     Parallel transport: automatic            │
│     No coordinate singularities             │
│     (θ,φ,β) derived on demand               │
├─────────────────────────────────────────────┤
│              GLOBE LAYER                     │
│     Metric tensor                           │
│     Great circle distance                   │
│     Bearing computation                     │
│     Horizon calculation                     │
│     Terrain lookup via (θ,φ)                │
├─────────────────────────────────────────────┤
│              LOCAL LAYER                     │
│     Tangent-plane Euclidean frames          │
│     Object placement within buildings       │
│     Local physics (jumping, throwing)       │
│     Gravity as a local vertical force       │
├─────────────────────────────────────────────┤
│            RENDER BRIDGE                     │
│     SO(3) matrix → view matrix for GPU      │
│     One-way, mechanical conversion          │
│     Written once, never revisited           │
├─────────────────────────────────────────────┤
│                  GPU                         │
│     Standard perspective projection         │
│     Knows nothing about the sphere          │
└─────────────────────────────────────────────┘
```

---

## 21. What Is NOT Defined (Intentionally)

- No "world space coordinates" beyond (θ, φ, h).
- No transformation matrices between "world space" and "local space" — there is no world space in the Euclidean sense.
- No global up vector.
- No origin point.
- No global axes.

The only global truths are R, the axis, the prime meridian, and the metric. Everything else is local.

---

## 22. Implementation Status

**2D Proof of Concept (complete):** A browser-based engine implementing S¹ × ℝ (circle × line) with a dual-view renderer (intrinsic side-scroller + extrinsic circle view). Located in `2D/`. Includes a Mario-style platformer demo exercising all engine features: movement, wrapping, geodesic projectiles, gravity, collision, zoom revealing periodicity. Note: S¹ is flat (zero curvature), so this prototype validates the rendering architecture and design patterns, not the curved geometry. The curved geometry starts in 3D.

**3D Prototype (in progress):** A browser-based engine implementing S² × ℝ using Three.js. Located in `3D/`. First-person camera on a sphere with SO(3) state, ground mesh generation, dual-view rendering, and markers for circumnavigation proof. This is where the non-trivial geometry lives — parallel transport, metric correction, pole singularity handling, great circle geodesics.

---

## 23. Summary

Three choices found the world: a radius, an axis, a reference line.

One object encodes the geometry: the metric tensor.

One matrix encodes the state: an SO(3) rotation matrix plus a height scalar.

Developers speak planet language: latitude, longitude, bearing, arc distance. The engine translates silently to singularity-free matrix rotations under the hood.

The world's geometry is S² × ℝ — not a planet in Euclidean space, but a universe with its own rules. It feels like Earth locally. It wraps like nothing that has ever existed.

The sphere is not in the scene. The sphere is the scene.
