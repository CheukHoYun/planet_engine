# Planet Engine

A set of small browser prototypes that ask: *what would it be like to walk on a world whose geometry is simple but not Euclidean?*

The core idea is a split view. The **left panel** shows the world the way an inhabitant sees it — flat, normal, the geometry it actually has. The **right panel** is the same world re-described in Euclidean 3D so an outsider can see its global shape (a circle, a sphere, a Möbius strip). The right view is a translation, not a camera looking down from outside; the curvature on the right exists only because we forced an intrinsic world into a foreign coordinate system. Walk far enough on the left and you come back where you started — that's the world's topology made visible.

## Live demos

- **2D — S¹ × ℝ:** [open](https://cheukhoyun.github.io/planet_engine/2D/) · [source](./2D)
  The world is a circle crossed with a height line.
- **3D — S² × ℝ:** [open](https://cheukhoyun.github.io/planet_engine/3D/) · [source](./3D)
  The world is a sphere crossed with a height line.
- **Möbius:** [open](https://cheukhoyun.github.io/planet_engine/mobius/) · [source](./mobius)
  A Möbius strip world. Walking around once turns you upside-down relative to where you started.

Each prototype is a single HTML file. Open it in a browser; no build step.

## Write-up

[The Planet Engine — leonsnotes.ca](https://leonsnotes.ca/2026/05/04/the-planet-engine/)
