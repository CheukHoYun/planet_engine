# The Intrinsic Möbius Engine

## Adapting the S² × ℝ Engine to a Non-Orientable World

---

## 1. What Changes, What Doesn't

The sphere engine's architecture — intrinsic coordinates, metric tensor, exponential map, tangent-plane rendering, universal cover preimages — carries over intact. The Möbius strip is a different manifold, but the engine's philosophy ("the surface is the space") applies without modification.

What changes:

- The manifold is **flat** (zero Gaussian curvature), so geodesics are straight lines, parallel transport is trivial, and the exponential map collapses to linear arithmetic. The engine becomes simpler, not harder.
- The manifold is **non-orientable**. This is the payoff — the thing the sphere couldn't do. An entity that walks a full loop returns to its starting position with its left and right reversed.
- The manifold has a **boundary** (the edges of the strip). Entities can fall off, or you can make the edges walls.
- The state representation changes from SO(3) to **(u, v, β, h)** — the coordinate singularity at the poles is gone, replaced by the identification rule.

What doesn't change:

- The dual-view architecture (intrinsic tangent plane + extrinsic embedding).
- The entity system (preimage clones, per-entity uniforms).
- The rendering pipeline (flat ground mesh + exponential-map shader for intrinsic; parametric embedding shader for extrinsic).
- The principle that the GPU sees Euclidean coordinates only at the final step.

---

## 2. The Manifold

The world is a **Möbius band crossed with a line**: M × ℝ.

The Möbius band M is defined as a quotient of the Euclidean plane. Start with the infinite strip ℝ × [−W, W] and impose a single identification:

```
(u, v)  ~  (u + L, −v)
```

- **u** is the coordinate "around" the strip. Range: all of ℝ, but with period L under the identification (with a twist).
- **v** is the coordinate "across" the strip. Range: [−W, W]. The edges v = ±W are the boundary of the strip.
- **L = 2πR** is the length of the central circle (v = 0). R is the sole geometric parameter, just like the sphere engine.
- **W** is the half-width of the strip. A second parameter the sphere didn't have.

The height coordinate h works almost identically to the sphere engine — Euclidean, measured in meters, independent of the surface geometry — with one subtlety we'll see in §4: h flips sign with v at the identification, because the embedding's surface normal flips after one lap.

```
World:
    R     — radius of the central circle (meters)
    W     — half-width of the strip (meters)
    L     — circumference = 2πR (derived)
```

---

## 3. The Metric Tensor

```
g = | 1   0 |
    | 0   1 |
```

That's it. The Möbius strip with this identification is **flat** — locally indistinguishable from ℝ². The metric tensor is the identity everywhere. No cos²φ factor, no coordinate-dependent scaling, no curvature.

This means:

- Distances are Euclidean: d = √(Δu² + Δv²).
- Angles are Euclidean.
- Areas are Euclidean.
- The Christoffel symbols are all zero.
- Geodesics are straight lines (in (u, v) coordinates).
- Parallel transport is trivial: just carry your vector along.

All the geometric complexity of the sphere engine — Rodrigues rotation, great circle distance, metric correction, Christoffel symbols — disappears. The only interesting structure is **topological**: the identification rule.

> **Convention:** we use (u, v) as the unit-speed orthonormal chart, so `u` is *arc length* along the centerline and `v` is *arc length* across the strip. Both are in meters. With this convention, walking-speed in the engine is `m/s` directly, and `L = 2πR` is consistent with `u`-velocity = `m/s` without an `R`-divisor. The slider that varies `R` must NOT rescale the unit of `u` — it grows the period `L` instead.

---

## 4. The Identification Rule

The identification rule for the **2D surface** is:

```
(u, v)  ~  (u + L, −v)
```

But the engine lives in three dimensions: $M \times \mathbb{R}$ with height. The full identification on the 3-manifold has to also flip h:

```
(u, v, h)  ~  (u + L, −v, −h)
```

This is the **deck transformation** T of the universal cover. It generates the fundamental group π₁(M) ≅ ℤ. Its powers give all preimages of any point:

```
Tⁿ: (u, v, h) → (u + nL, (−1)ⁿ · v, (−1)ⁿ · h)
```

For even n, both v and h are unchanged. For odd n, both flip. This is the non-orientability: after an odd number of laps, your cross-direction AND your "up" direction are reversed in the strip's coordinate system.

### Why h flips

The Möbius strip is non-orientable: there is no globally consistent surface normal. The standard ribbon embedding (§11) picks a smooth normal at every point, but that normal **reverses direction after one lap**. So if h is interpreted as "distance along the surface normal," then on the other side of the gluing the same physical position has h with its sign flipped, because the meaning of "+h" itself flipped.

The self-consistency check that fixes the engine: the embedding evaluates `worldPos = surface(u, v) + h · n(u, v)`. Under the deck transformation, both `n` and `h` flip — so

```
(−h) · (−n) = h · n
```

The product is unchanged. The 3D position is the same. The flip is purely a coordinate change.

### What this means physically

The inhabitant doesn't move. They are always 1.7 m above the ground from their own perspective. The flip is bookkeeping. After one lap, the *coordinate label* for their height is now −1.7, because the coordinate system's "+h" direction has reversed. `|h|` is the physical height; `sign(h)` is the wrap parity.

For the **intrinsic view**, none of this matters: the camera height is `|h|`, and gravity always pulls toward `h = 0`. The sign only exists to keep the **extrinsic embedding** consistent (no visual snap when crossing a wrap boundary).

### Effect on tangent vectors

The 2D part of the deck transformation maps:

```
∂/∂u  →  ∂/∂u       (the "around" direction is preserved)
∂/∂v  →  −∂/∂v      (the "across" direction is flipped)
```

So a direction vector (cos β, sin β) in the (u, v) basis maps to (cos β, −sin β) = (cos(−β), sin(−β)).

**Bearing transforms as: β → −β** (equivalently, 2π − β).

This is the intrinsic, coordinate-free manifestation of non-orientability. No concept of "sides." Just: odd laps negate the cross-component of every tangent vector, and the height coordinate flips along with it.

---

## 5. State Representation

### Sphere engine (for comparison)

```
State = SO(3) matrix M  +  scalar h
```

The SO(3) matrix encodes position (the "up" column = point on unit sphere) and orientation (the "right" and "forward" columns) simultaneously, with no coordinate singularities.

### Möbius engine

```
State = (u, v, β, h, vh)
```

- **u ∈ [0, L)** — position along the strip (the non-contractible direction); periodic
- **v ∈ [−W, W]** — position across the strip
- **β ∈ [0, 2π)** — bearing (0 = +u direction, π/2 = +v direction)
- **h ∈ ℝ** — *signed* height in the current coordinate patch; physical height is `|h|`
- **vh ∈ ℝ** — vertical velocity in the same patch

No SO(3) matrix needed. The Möbius strip has no coordinate singularities (unlike the sphere's poles), so raw coordinates work everywhere.

### Normalization (Fix A)

After any state update, normalize u into the canonical range [0, L) and apply the deck transformation each time u crosses a period boundary:

```javascript
function normalize(state) {
    while (state.u >= L) {
        state.u -= L;
        state.v   = -state.v;
        state.beta = -state.beta;
        state.h   = -state.h;       // <-- the h-flip
        state.vh  = -state.vh;      // <-- and its derivative
    }
    while (state.u < 0) {
        state.u += L;
        state.v   = -state.v;
        state.beta = -state.beta;
        state.h   = -state.h;
        state.vh  = -state.vh;
    }
    state.v = clamp(state.v, -W, W);
}
```

Every time u crosses a period boundary, **all four signed quantities — v, β, h, vh — flip together**. This is the deck transformation acting on position, tangent vector, and height fiber simultaneously.

### Wrap parity for "up at h = 0"

When the player is grounded (`h = 0`), the sign of h gives no information about which face of the strip they're on. For two cases this matters:

- **Jumping.** "Up" must impart positive vh in the *current* patch. The engine needs to know which side it's on.
- **Mesh placement.** The player's mesh extends in the +y direction in its local frame; that direction must point outward from the strip in extrinsic space.

Both are solved by tracking a single bit alongside the state:

```javascript
state.parity = +1;   // ±1; flipped on every wrap
```

`normalize()` flips `parity` together with the other quantities. When `h ≠ 0`, `sign(h) === state.parity`. When `h === 0`, `state.parity` carries the same information.

Jumping: `state.vh = state.parity * JUMP_VEL`.

Gravity (general form, valid at h ≠ 0 and h = 0): `state.vh -= state.parity * GRAVITY * dt`. Equivalently, write it as `state.vh -= sign-of-current-patch * GRAVITY * dt` where the sign is just `state.parity`.

---

## 6. Movement

Since the metric is flat, movement is addition:

```javascript
function move(state, dt) {
    state.u += WALK_SPEED * Math.cos(state.beta) * dt;
    state.v += WALK_SPEED * Math.sin(state.beta) * dt;
    normalize(state);
}
```

Compare to the sphere engine, where movement requires rotating an SO(3) matrix (`world.frame.rotateAround(right, −speed·dt/R)`). Here it's just two additions. The geometry is flat — no curvature corrections.

**Turning** is equally trivial:

```javascript
state.beta += TURN_SPEED * dt;
```

No parallel transport needed. The Christoffel symbols are zero, so the direction a gyroscope points doesn't change as you move. (On the sphere, parallel transport along a closed loop rotates the direction — that's holonomy. On the flat Möbius strip, there's no holonomy for small loops. But there IS a holonomy-like effect for the non-contractible loop: the β → −β flip.)

---

## 7. The Exponential Map

The intrinsic fragment shader needs the exponential map: given a point on the tangent plane at distance (arcX, arcZ) from the player, what point on the manifold does it correspond to?

On the sphere, this requires Rodrigues rotation — the most complex piece of the intrinsic shader. On the Möbius strip, it's just:

```glsl
// Fragment shader: exponential map for the Möbius strip.
// arcX = rightward in tangent plane, arcZ = forward (= −vTangentPos.z).
// Bearing convention: forward in (u,v) = (cos β, sin β); right = (sin β, −cos β).
float du = arcX * sin(uPlayerBeta) + arcZ * cos(uPlayerBeta);
float dv = -arcX * cos(uPlayerBeta) + arcZ * sin(uPlayerBeta);
float hitU = uPlayerU + du;
float hitV = uPlayerV + dv;

// Apply identification: reduce hitU to [0, L). On odd-wrap fragments, flip v.
float wraps = floor(hitU / uL);
hitU -= wraps * uL;
if (mod(abs(wraps), 2.0) > 0.5) hitV = -hitV;
```

This replaces the entire Rodrigues rotation block in the sphere engine's intrinsic fragment shader. Flat geometry means the exponential map is a straight line in (u, v) space plus the identification.

---

## 8. Parallel Transport

On the sphere, parallel transport of a direction from point A to point B along a geodesic requires computing the angular defect — the direction rotates relative to the local coordinate axes because the Christoffel symbols are nonzero.

On the Möbius strip: **parallel transport along any path that doesn't cross the identification is trivial** (the vector is unchanged). The only nontrivial effect is topological: a vector parallel-transported around the full non-contractible loop returns with its v-component negated.

In the engine, this is handled automatically by the normalization step. When the player's u wraps past a period boundary, β flips — that IS the parallel transport.

For entity rendering: when computing how an entity's facing direction appears in the player's tangent frame, the number of identification crossings between them determines whether the apparent facing is flipped:

```javascript
// How many full wraps separate entity from player?
const deltaU = entity.u - player.u;
const wraps  = Math.round(deltaU / L);
const apparentBeta = (wraps % 2 === 0) ? entity.beta : -entity.beta;
```

---

## 9. Universal Cover and Preimages

The universal cover of the Möbius strip is the infinite strip ℝ × [−W, W] — the same rectangle we started with, but without the identification. Lifted to M × ℝ, the cover is ℝ × [−W, W] × ℝ. The deck transformations are:

```
Tⁿ: (u, v, h) → (u + nL,  (−1)ⁿ · v,  (−1)ⁿ · h)       for all n ∈ ℤ
```

An entity at position (u₀, v₀, h₀) has preimages at:

```
(u₀ + nL,  v₀,  h₀)         for n even
(u₀ + nL, −v₀, −h₀)         for n odd
```

In the player's tangent plane, each preimage appears at:

```javascript
const sign = (n % 2 === 0) ? 1 : -1;
const preU =  entity.u + n * L;
const preV =  sign * entity.v;
const preH =  sign * entity.h;

// Now project (preU, preV, preH) into the camera's tangent frame:
const du = preU - camera.u;
const dv = preV - camera.v;
const cosB = Math.cos(camera.beta);
const sinB = Math.sin(camera.beta);
const xLocal =  du * sinB - dv * cosB;
const zLocal = -(du * cosB + dv * sinB);
const yLocal =  preH;

placePreimage(xLocal, yLocal, zLocal, sign /* mirror flag */);
```

Compare to the sphere engine, where preimages lie along a great circle and distances are arc lengths. Here they lie along a straight line in the u-direction, evenly spaced by L, alternately flipped in v AND h. Much simpler.

**Key difference from the sphere**: On the sphere, preimages lie along the unique great-circle path between camera and entity. On the Möbius strip, preimages are a 1D arithmetic progression in u, with sign-flipping on odd wraps for both cross-direction and height.

---

## 10. Boundary Handling

The sphere has no boundary. The Möbius strip does: the edges at v = ±W.

Options:

1. **Hard wall**: clamp v to [−W, W]. Entities bounce or stop at the edge.
2. **Fall-off**: entities with |v| > W are removed or respawned. Adds a gameplay element the sphere didn't have.
3. **Soft boundary**: slow movement near the edges (a potential function).
4. **No boundary** (open Möbius band): let v range over all of ℝ. The strip is infinitely wide. Non-orientability is preserved but there's no edge. This is the cleanest topological choice but loses the "strip" feeling.

**Choice for this prototype: option 1 (hard wall).** Render a thin colored vertical band at v = ±W in both views so the boundary is visible. Fall-off needs a respawn system the engine doesn't have; save it for later.

---

## 11. Extrinsic Embedding

For the right-panel "god view," embed the Möbius strip in ℝ³ using the standard parametrization:

```
x(u, v) = (R + v · cos(u/2)) · cos(u)
y(u, v) = (R + v · cos(u/2)) · sin(u)
z(u, v) = v · sin(u/2)
```

where u ∈ [0, 2π) and v ∈ [−W, W]. **Note**: this embedding uses `u` as an angle (radians), so the engine's arc-length `u` enters the embedding via `u · 2π / L`. The embedding itself has period 2π in this angle; one player lap is one trip around it.

This embeds the strip as a ruled surface — a surface swept out by a line segment rotating half a turn as it revolves around a circle.

### The self-consistency check (why Fix A works)

At u + 2π:
- `cos((u+2π)/2) = −cos(u/2)`
- `sin((u+2π)/2) = −sin(u/2)`

so `surface(u + 2π, v) = surface(u, −v)` — fine, that's the gluing of the surface. But for the **surface normal** (computed from `∂r/∂u × ∂r/∂v`), the same identities give

```
n(u + 2π, v) = −n(u, −v)
```

The normal flips sign across the gluing. So the embedded position `worldPos = surface + h · n` satisfies

```
worldPos(u, v, h) = worldPos(u + 2π, −v, −h)
```

Both `n` and `h` flip, and `(−h) · (−n) = h · n`. The product is unchanged. The 3D position is the same. **This is exactly the relation that makes Fix A work end-to-end** — the player crossing a wrap doesn't snap because the new strip-coords give the same embedded 3D position.

### Embedding vertex shader

Replace the sphere engine's Rodrigues-based embedding with:

```glsl
uniform float uObjU;        // entity's u coordinate
uniform float uObjV;        // entity's v coordinate
uniform float uObjH;        // entity's signed height
uniform float uObjBeta;     // entity's bearing
uniform float uR;           // world radius
uniform float uL;           // circumference = 2π · uR

void main() {
    // Local vertex in the entity's tangent frame:
    //   xLocal = offset in entity's right direction
    //   yLocal = height offset along the surface normal
    //   zLocal = offset in entity's forward direction
    vec4 lp = modelMatrix * vec4(position, 1.0);
    float xLocal = lp.x;
    float yLocal = lp.y;
    float zLocal = lp.z;

    // Map local offset to surface coordinates
    float hitU = uObjU + xLocal * sin(uObjBeta) + zLocal * cos(uObjBeta);
    float hitV = uObjV + xLocal * cos(uObjBeta) - zLocal * sin(uObjBeta);
    float h    = uObjH + yLocal;

    // Embed (hitU, hitV, h) into ℝ³
    float angle  = hitU * 6.283185307 / uL;
    float halfA  = angle * 0.5;
    float cosU   = cos(angle);
    float sinU   = sin(angle);
    float cosH   = cos(halfA);
    float sinH   = sin(halfA);

    float rEff = uR + hitV * cosH;
    vec3 surfacePos = vec3(rEff * cosU, rEff * sinU, hitV * sinH);

    // Surface normal = ∂r/∂u × ∂r/∂v, normalized
    vec3 ddu = vec3(
        -rEff * sinU - 0.5 * hitV * sinH * cosU,
         rEff * cosU - 0.5 * hitV * sinH * sinU,
         0.5 * hitV * cosH
    );
    vec3 ddv = vec3(cosH * cosU, cosH * sinU, sinH);
    vec3 normal = normalize(cross(ddu, ddv));

    vec3 worldPos = surfacePos + h * normal;
    gl_Position = projectionMatrix * viewMatrix * vec4(worldPos, 1.0);
}
```

### Building the extrinsic mesh

Replace `buildExtrinsicSphere` with:

```javascript
function buildExtrinsicMobius(R, W, segmentsU, segmentsV) {
    const positions = [];
    const indices = [];

    for (let j = 0; j <= segmentsV; j++) {
        const v = -W + (2 * W) * (j / segmentsV);
        for (let i = 0; i <= segmentsU; i++) {
            const u = (2 * Math.PI) * (i / segmentsU);
            const halfU = u / 2;
            const rEff = R + v * Math.cos(halfU);
            positions.push(
                rEff * Math.cos(u),
                rEff * Math.sin(u),
                v * Math.sin(halfU)
            );
        }
    }

    for (let j = 0; j < segmentsV; j++) {
        for (let i = 0; i < segmentsU; i++) {
            const a = j * (segmentsU + 1) + i;
            const b = a + 1;
            const c = a + (segmentsU + 1);
            const d = c + 1;
            indices.push(a, b, c, b, d, c);
        }
    }

    const geo = new THREE.BufferGeometry();
    geo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
    geo.setIndex(indices);
    geo.computeVertexNormals();
    return geo;
}
```

> **Material side:** the strip mesh AND every entity material that uses the embedding shader **must** use `THREE.DoubleSide`. The Möbius surface has no consistent front/back; some triangles face "outward" and some "inward" relative to the camera as you orbit, and the entity meshes are mirrored on odd preimages (§15) which reverses winding. `FrontSide` would silently cull half the geometry. Add a one-line comment at every material declaration so this isn't accidentally regressed.

### Edge bands

Render two thin colored bands at v = ±W as separate ribbons — same embedding, with v fixed at ±W and a small thickness in h. Use saturated red on the +W edge and saturated blue on the −W edge so the orientation flip is unmistakable as you orbit.

---

## 12. Intrinsic Ground Shader

The sphere engine's intrinsic fragment shader does:

1. Take the tangent-plane position (arcX, arcZ)
2. Compute the exponential map via Rodrigues rotation → hitPoint on the unit sphere
3. Extract (θ, φ) from hitPoint
4. Draw grid lines from (θ, φ)

The Möbius engine's intrinsic fragment shader does:

1. Take the tangent-plane position (arcX, arcZ)
2. Compute the exponential map via **addition** → (hitU, hitV) on the strip
3. Apply the identification to reduce (hitU, hitV) to canonical range
4. Draw grid lines from (hitU, hitV) and a v-gradient that visualizes the flip

```glsl
precision highp float;
uniform float uPlayerU;
uniform float uPlayerV;
uniform float uPlayerBeta;
uniform float uR;
uniform float uW;
uniform float uL;
varying vec3 vTangentPos;

void main() {
    float arcX = vTangentPos.x;
    float arcZ = -vTangentPos.z;

    // Exponential map: straight line in (u,v).
    float cosBeta = cos(uPlayerBeta);
    float sinBeta = sin(uPlayerBeta);
    float hitU = uPlayerU + arcX * sinBeta + arcZ * cosBeta;
    float hitV = uPlayerV - arcX * cosBeta + arcZ * sinBeta;

    // Identification: reduce hitU to [0, L); flip hitV on odd-wrap fragments.
    float wraps = floor(hitU / uL);
    hitU -= wraps * uL;
    if (mod(abs(wraps), 2.0) > 0.5) hitV = -hitV;

    // Grid lines (u every L/20 m; v every W/10 m)
    float gridU = smoothstep(0.94, 0.96, abs(cos(hitU * 6.283185 * 10.0 / uL)));
    float gridV = smoothstep(0.94, 0.96, abs(cos(hitV * 6.283185 *  5.0 / uW)));

    // Linear blue→red gradient across v. After one wrap, blue and red swap.
    // Endpoints match the 3D engine's polar palette so the prototypes feel
    // like the same product.
    float t = (hitV + uW) / (2.0 * uW);
    vec3 cN = vec3(0.92, 0.10, 0.10);   // +v edge
    vec3 cS = vec3(0.10, 0.18, 0.92);   // -v edge
    vec3 baseColor = mix(cS, cN, t);

    // Outside the strip: transparent (let the sky/void show through).
    if (abs(hitV) > uW) {
        gl_FragColor = vec4(0.0, 0.0, 0.0, 0.0);
        return;
    }

    float grid = max(gridU, gridV);
    vec3 color = mix(baseColor, vec3(0.05), grid);

    // Thin highlight at the v = ±W boundary, so the player can see the edge.
    float edgeBand = smoothstep(0.97, 1.0, abs(hitV) / uW);
    color = mix(color, vec3(0.95), edgeBand * 0.6);

    gl_FragColor = vec4(color, 1.0);
}
```

The blue-to-red gradient across v makes the non-orientability visible: after one full lap, blue and red swap sides.

---

## 13. What the Inhabitant Experiences

An inhabitant of M × ℝ lives in a world that is locally flat — no curvature effects at all. But:

1. **The world is periodic but twisted.** Walk far enough in one direction and you return to your starting point. But everything is mirror-reflected. Text reads backward. Your left hand is now your right hand. The blue edge is now the red edge.

2. **Walk another lap and everything is normal again.** The double-cover of the Möbius strip is a cylinder. Two laps brings you back to the original orientation.

3. **You can see yourself.** At view distance L, you see a mirror image of yourself. At distance 2L, you see yourself normally. At 3L, mirrored again. The odd preimages are all reflected; the even ones are not.

4. **The world has edges.** Walk sideways and you reach the boundary. The sphere had no boundary. This adds a physical constraint — a wall.

5. **No poles, no equator.** The sphere had special points (poles with coordinate singularities) and a special curve (the equator with maximal circumference). The Möbius strip has no distinguished points at all. Every point on the central circle (v = 0) is identical. Every point off-center has a partner under the identification.

6. **Parallel transport is globally inconsistent.** Carry a gyroscope around the central loop. When you return, it points in the opposite cross-direction. This is the intrinsic, observer-detectable consequence of non-orientability. Not "two sides" — just a vector that can't be globally defined.

7. **Your height never changes from your perspective.** You're always 1.7 m above the ground. Gravity always pulls "down." Jumping always sends you "up." But in the engine's coordinate patch the *label* of your height flips sign each lap. The sign is bookkeeping; you don't feel it.

---

## 14. Coordinate System Comparison

| Concept | Sphere engine | Möbius engine |
|---|---|---|
| Manifold | S² | Möbius band |
| Topology | Orientable, no boundary, π₁ = 0 | Non-orientable, has boundary, π₁ = ℤ |
| Curvature | K = 1/R² (positive, constant) | K = 0 (flat) |
| Coordinates | (θ, φ) with pole singularities | (u, v) with no singularities |
| State | SO(3) matrix + h | (u, v, β, h, vh) + parity |
| Metric tensor | diag(R²cos²φ, R²) | diag(1, 1) |
| Movement | SO(3) rotation | Vector addition |
| Exponential map | Rodrigues rotation | Straight line + identification |
| Geodesics | Great circles | Straight lines |
| Parallel transport | Nontrivial (holonomy) | Trivial locally; β-flip on full loop |
| Identification | (θ, φ) ~ (θ, φ) (simply connected) | (u, v, h) ~ (u + L, −v, −h) |
| Preimage pattern | Along a great circle, arc-spaced | Along a line, L-spaced, alternately v-flipped AND h-flipped |
| Boundary | None | v = ±W (edges of the strip) |
| Universal cover | ℝ² (the plane) | ℝ × [−W, W] (infinite strip) |

---

## 15. Mirror Rendering for Odd Preimages

This is the most important rendering detail. On the sphere, all preimages are identical copies — same mesh, same orientation. On the Möbius strip, **odd preimages are mirror-reflected in BOTH the cross-direction and the height-direction**.

In the sphere engine, every preimage clone uses the same quaternion:
```javascript
c.quaternion.setFromRotationMatrix(_entMat);
```

In the Möbius engine, odd preimages must flip the mesh on two axes:
```javascript
if (isOddWrap) {
    // Mirror the entity mesh:
    //   x flips  →  the v-direction reflection (left ↔ right)
    //   y flips  →  the h-direction reflection (head ↔ feet, i.e. the
    //               outward-normal direction reverses)
    c.scale.set(-1, -1, 1);
} else {
    c.scale.set(1, 1, 1);
}
```

This is what makes the player see a mirror image of themselves at distance L. The asymmetric "R" on the player's belt reads backward. Text on signs reads backward. Tall structures appear on the opposite side of the strip.

> Mirroring with `scale.set(-1, -1, 1)` reverses triangle winding (a 180° rotation of normals about Z, then a Z-flip cancels — the determinant is `(-1)(-1)(1) = +1`, so winding is *preserved*). Combined with `DoubleSide` materials this is robust regardless. Worth a comment at the call site.

Without this, the non-orientability is invisible.

---

## 16. The `projectToTangent` Replacement

The sphere engine's `projectToTangent(upWorld, h, frame, out)` is the core function that places an entity in the camera's tangent plane. It uses arc distance and bearing computed from dot products with the SO(3) frame vectors.

On the Möbius strip, this collapses to coordinate subtraction plus a bearing rotation:

```javascript
function projectToTangent(entityU, entityV, entityH, cameraU, cameraV, cameraBeta, out) {
    const du = entityU - cameraU;
    const dv = entityV - cameraV;

    // Camera's tangent +X = "right", -Z = "forward" (Three.js convention).
    // forward in (u,v) = (cos β, sin β)
    // right   in (u,v) = (sin β, −cos β)
    const cosB = Math.cos(cameraBeta);
    const sinB = Math.sin(cameraBeta);

    const xLocal =  du * sinB - dv * cosB;     // rightward
    const zLocal = -(du * cosB + dv * sinB);   // Three.js: forward = -Z

    out.set(xLocal, entityH, zLocal);
    return out;
}
```

No arc distance, no dot products, no Rodrigues. Just a 2D rotation.

---

## 17. Camera System

The sphere engine has `cameraSurfaceFrame` — a separate SO(3) frame for the camera's surface projection, offset from the player by the orbit distance. This is needed on the sphere because offsetting a position requires rotating the frame (non-trivial on curved geometry).

On the flat Möbius strip, the camera's surface position is just a (u, v) offset from the player:

```javascript
function updateCameraSurface(world) {
    // Camera is behind the player at horizontal distance ORBIT_RADIUS · cos(pitch).
    const horizDist = ORBIT_RADIUS * Math.cos(world.pitch);

    // "Behind" = opposite of the player's forward direction, plus the
    // user's free azimuth offset around the player.
    const camBearing = world.beta + Math.PI + world.cameraAzim;

    cameraSurface.u    = world.u + horizDist * Math.cos(camBearing);
    cameraSurface.v    = world.v + horizDist * Math.sin(camBearing);
    cameraSurface.beta = world.beta + world.cameraAzim;
    cameraSurface.h    = world.h;
    cameraSurface.parity = world.parity;

    // Normalize (handles identification if camera crosses a period boundary
    // independently of the player). normalize() will flip h, vh, parity if
    // the camera's u wraps; that's correct, the camera is just another
    // point on the manifold.
    normalize(cameraSurface);
}
```

The tangent plane is then centered on `cameraSurface`, and all entities (including the player) are projected relative to it.

---

## 18. Entity Orientation in the Tangent Frame

The sphere engine computes entity yaw by projecting the entity's world-forward vector onto the camera's tangent plane using dot products. On the Möbius strip:

```javascript
// Wrap-count between this preimage and the camera.
const isOddWrap = (n + parityDiff) % 2 !== 0;

// Apparent bearing in the player's frame:
//   even wrap → entity.beta − camera.beta
//   odd wrap  → −entity.beta − camera.beta  (deck transform negates β)
let relBeta = isOddWrap
    ? (-entity.beta - camera.beta)
    : ( entity.beta - camera.beta);

mesh.rotation.y = -relBeta;
```

`parityDiff` accounts for the camera's own parity vs. the entity's stored bearing.

---

## 19. Bearing Convention and Sign Reference

To keep the shaders and JS consistent, fix this convention everywhere:

```
β = 0      →  facing +u direction  (around the strip)
β = π/2    →  facing +v direction  (across the strip, toward +W edge)
β = π      →  facing −u direction
β = 3π/2   →  facing −v direction  (toward −W edge)
```

Tangent frame mapping (Three.js: +X = right, −Z = forward):

```
forward in (u,v) = (cos β, sin β)
right   in (u,v) = (sin β, −cos β)

tangent X (rightward) =   du · sin β  −  dv · cos β
tangent Z (backward)  = −(du · cos β  +  dv · sin β)
```

The exponential map in the intrinsic shader inverts this:

```
du =  arcX · sin β + arcZ · cos β       (where arcZ = −vTangentPos.z)
dv = −arcX · cos β + arcZ · sin β
```

---

## 20. Implementation Checklist

### Replace

- [ ] `Frame` class (SO(3) matrix) → `State` object `{u, v, beta, h, vh, parity, grounded}`
- [ ] `frame.rotateAround(axis, angle)` → `state.u += ...; state.v += ...; normalize(state);`
- [ ] `projectToTangent(upWorld, h, frame, out)` → direct (Δu, Δv) computation with bearing rotation
- [ ] `setFrameFromUpForward(frame, up, fwd)` → `state.beta = atan2(fwd_v, fwd_u)`
- [ ] `buildExtrinsicSphere(R)` → `buildExtrinsicMobius(R, W, ...)`
- [ ] Extrinsic grid fragment shader (θ, φ grid + N/S labels + polar gradient) → (u, v grid + edge highlight + blue↔red v-gradient)
- [ ] Intrinsic fragment shader (Rodrigues exp map) → (linear exp map + identification)
- [ ] Extrinsic embedding vertex shader (unit sphere × (R+h)) → (Möbius parametric × normal offset)
- [ ] `SphereGeometry` → custom Möbius geometry, `DoubleSide` material everywhere
- [ ] Preimage loop: arc-spaced great-circle preimages → L-spaced line preimages with v-flip AND h-flip on odd n
- [ ] Odd-preimage mesh transform: `c.scale.set(-1, -1, 1)` (was identity on the sphere)
- [ ] Coordinate readout: (θ°, φ°) → (u m, v m) plus a parity badge (e.g., "side A" / "side B")

### Add

- [ ] Boundary clamp (`v ∈ [−W, W]`) + visible colored band at v = ±W in both views
- [ ] `W` parameter + slider (alongside the existing `R` slider)
- [ ] Parity tracking on the player state, flipped in `normalize()`
- [ ] Asymmetric demo entities (§22): "R" on player belt, signpost with readable text
- [ ] Gravity + jump using `parity`: `vh -= parity·GRAVITY·dt`, `vh = parity·JUMP_VEL`

### Remove

- [ ] Pole singularity handling (there are no poles)
- [ ] `cos²φ` metric correction
- [ ] N / S letter SDF rendering (no poles to label)
- [ ] Pole fade for longitude lines
- [ ] Polar saturated red/blue gradient (replaced by linear v-gradient — same endpoints, different domain)

---

## 21. The Surface Normal for the Embedding

At a point (u, v) on the Möbius strip, the tangent vectors in the embedding are:

```
∂r/∂u = (
    −(R + v·cos(u/2))·sin(u)  −  (v/2)·sin(u/2)·cos(u),
     (R + v·cos(u/2))·cos(u)  −  (v/2)·sin(u/2)·sin(u),
     (v/2)·cos(u/2)
)

∂r/∂v = (
    cos(u/2)·cos(u),
    cos(u/2)·sin(u),
    sin(u/2)
)
```

The **outward normal** is `n = normalize(∂r/∂u × ∂r/∂v)`. This normal is well-defined at every point of the embedded surface — the embedding itself picks a normal. But it is **not globally consistent**: if you follow the normal continuously around the strip, it flips direction after one lap. This is exactly the non-orientability, made visible in the embedding.

For the height offset in the extrinsic view: `worldPos = surfacePoint + h · n`.

The flip in `n` and the flip in `h` cancel — `(−h) · (−n) = h · n` — so the inhabitant's 3D position is the same on both sides of the gluing. That is the property that lets the engine carry h as a *signed* quantity through the deck transformation without snaps.

---

## 22. Orientation Visualization

To make non-orientability viscerally obvious, the ground shader uses a **blue-to-red gradient across v** (§12):

- v = −W → saturated blue
- v = 0 → purple (the midline)
- v = +W → saturated red

After one full lap (u increases by L), the identification maps v → −v. Blue becomes red and red becomes blue. The player sees the colors swap without any discontinuity — the swap is gradual from their perspective, but the result after a full loop is unmistakable.

For entities, two demo pieces make the flip undeniable on day one:

### "R" on the player's belt

The existing 3D player has a pink belt for visual reference. Stamp a flat asymmetric letter — capital "R" — onto the +v side of the belt (the side that maps to "the right of facing-forward" in the convention above). Use a contrasting color (white or yellow) so it's legible from behind. After one lap, the player passes the camera with the "R" reading backward — this is the most direct observable consequence of non-orientability.

### Signpost entity

A new entity kind: a thin tall board, mounted vertically, with one face painted with readable text (e.g., the word **"FORWARD"** or **"RIGHT"**) and the other face left blank or painted a contrasting solid color. Place one signpost at `v ≈ +W/2` near `u = 0`. The player walks past it once normally — text is legible. Walks past it on the next lap (i.e., its odd-preimage) — text reads mirrored. Walks past it again — readable. Two laps to read normally; one to read backward. A textbook-level demonstration with no math required.

---

## 23. Summary of the Math

Three operations capture the entire geometry:

**1. Movement** — flat, trivial:
```
u += speed · cos(β) · dt
v += speed · sin(β) · dt
```

**2. Identification** — the topology:
```
When u leaves [0, L):   u mod L,  v → −v,  β → −β,  h → −h,  vh → −vh,  parity → −parity
```

**3. Embedding** — for the GPU:
```
x = (R + v·cos(u/2)) · cos(u)
y = (R + v·cos(u/2)) · sin(u)
z = v · sin(u/2)
worldPos = (x, y, z) + h · normal(u, v)
```

Everything else — distance, direction, preimages, parallel transport, the exponential map — follows from (1) and (2). The embedding (3) is used only for the extrinsic view.

The **single non-trivial fact** that holds all of this together is the embedding's self-consistency under the deck transformation:

```
(−h) · (−n) = h · n
```

The Möbius strip's non-orientability shows up as paired sign flips in both the height fiber and the surface normal. They cancel exactly. The player crosses a wrap and the world's 3D representation is unchanged — only the coordinate labels rotate.

The manifold's geometry is simpler than the sphere. Its topology is stranger. That's the trade.
