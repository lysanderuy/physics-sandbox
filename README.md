# Physics Sandbox

A 2D physics demo in C++ and OpenGL, built on Box2D. Shapes spawn at the mouse
and roll over chain terrain, ride a moving platform, and knock a rope and a
pendulum around. It is one world that touches all three Box2D modules: Common,
Collision and Dynamics. `src/main.cpp` is built up one checkpoint at a time, so
each commit in the history adds one concept.

GLFW, GLAD and Box2D are vendored, and window and input handling live in
`src/core/`. Nothing needs to be installed beyond the compiler.

## Prerequisites

- **MinGW-w64 via MSYS2**. Install from [msys2.org](https://www.msys2.org/),
  then install the `mingw-w64-x86_64-gcc` toolchain from the MSYS2 shell.
- **VS Code**.
- The MinGW `bin` folder (e.g. `C:\msys64\mingw64\bin`) on your system `PATH`.
  The build tasks call `g++` and `ar` by name, and the compiler's own child
  processes find their DLLs through the system `PATH`, so without this entry
  the build can fail silently.

## Build and run

1. Open the folder in VS Code.
2. Run the `Build Main` task (`Ctrl+Shift+B`). The first build also compiles
   Box2D into `lib/libbox2d.a`, which takes about 40 seconds. Later builds
   skip that step.
3. Run `main.exe` from the repo folder.

## Controls

- **Spawn:** hold the left mouse button. The selected shape appears at the
  cursor, at most one every quarter second.
- **Shape:** `1` box, `2` circle, `3` triangle, `4` pentagon, `5` hexagon.
- **AABB:** `B` shows or hides the axis-aligned bounding box of each shape.
- `Esc` closes the window.

The window title shows the selected shape and the world's body, contact and
joint counts.

## How it works

The window is 800x600 pixels with the origin at the top left and y pointing
down. Box2D works in meters with y pointing up, and 32 pixels make one meter
(`kPixelsPerMeter`). The world origin is the bottom left corner of the window.

The simulation runs at a fixed 60 Hz, driven by a time accumulator. A frame
longer than `kMaxFrameTime` is clamped, so a stall can never trigger a long
burst of steps.

Box2D positions are body centers and `SetAsBox` takes half extents.
`drawBoxBody` converts both to pixels and negates the angle, because rotation
counter-clockwise in Box2D is clockwise on a y-down screen. It draws the
platform, the rope and the anchors.

Every spawned item has the same material: density 1, friction 0.5 and
restitution 0.3. Its shape and color come from its entry in `kShapes`. Items
that fall below `kRemoveBelowY` meters are destroyed.

Box2D is three modules, and every `world.Step` runs all of them together.

### Common

`b2Transform` holds a body's position `p` and rotation `q`. `b2Mul(transform,
point)` turns a point in the body's local space into world space. `drawAxes`
uses it to draw each body's x axis in red and y axis in green, and `drawItem`
uses it to place shape vertices. `drawLine` is a thin box laid between two
points, built on `gfx::drawBox`.

`b2_maxPolygonVertices` (8) is a setting that caps the corners of a polygon.

The world owns every body, fixture and joint. `CreateFixture` copies the shape,
so shapes can live on the stack, and `DestroyBody` also frees the body's
fixtures and joints.

### Collision

Each fixture holds a shape, and `drawItem` checks `GetType()` to decide how to
draw it. It handles circles and polygons. The terrain is drawn from its own
points.

- **Circle:** a center and a radius, drawn as 24 line segments.
- **Polygon:** `SetAsBox` for the box. The other polygons place their corners
  on a circle with sin and cos and pass them to `Set`. The corners run
  counter-clockwise, the polygon must be convex, and the limit is eight.
  `Set` builds the convex hull itself, so corner order does not break it.
- **Chain:** a string of edges used as the terrain. A chain is one-sided: the
  solid side is right of the direction of travel, so the points run right to
  left to make the hilltops solid. The two extra points in `CreateChain` are
  the neighbors past each end. A shape spawned below the surface is on the
  open side, so it falls through and is destroyed.
- **AABB:** `fixture->GetAABB(0)` is the smallest box aligned to the world axes
  that holds the shape. It grows and shrinks as the shape rotates.

### Dynamics

- **Body types:** the terrain and the anchors are static, the items, rope links
  and bob are dynamic, and the platform is kinematic. A kinematic body is moved
  by its velocity and ignores forces. The main loop flips its velocity at
  `kPlatformMinX` and `kPlatformMaxX`.
- **Revolute joint:** pins two bodies to one point and lets them rotate. The
  rope is `kRopeLinks` links, each pinned to the one before it, and the first
  is pinned to a static anchor. `angularDamping` lets the rope settle.
- **Distance joint:** keeps two anchor points a fixed length apart. The bob
  swings on a 5 meter line and has no damping, so it keeps swinging.
- **World counts:** `GetBodyCount`, `GetContactCount` and `GetJointCount` are
  shown in the window title.

The values to tune are the constants at the top of `src/main.cpp`:
`kGravity`, `kPixelsPerMeter`, `kStepDt`, `kSpawnCooldown`, `kShapeSize`,
`kPolygonRadius`, `kPlatformSpeed` and `kRopeLinks`.

## Rebuild the Box2D library

`lib/libbox2d.a` is not committed, because a static C++ library built with one
GCC version can fail to link with another. Each machine builds its own. To
rebuild it after switching compilers or editing `src/box2d/`, run the
`Build Box2D library` task.

## Project structure

```
.vscode/tasks.json        build tasks
glfw3.dll                 GLFW runtime, must sit next to main.exe
lib/libglfw3dll.a         GLFW import library
src/
  main.cpp                world, shapes, joints, spawning, main loop
  draw.h/.cpp             gfx::drawBox and shader setup
  glad.c                  vendored GLAD loader
  core/                   GLFW context, window and keyboard input (gfx namespace)
  box2d/                  vendored Box2D v2.4.2 sources and its LICENSE
include/
  box2d/                  Box2D headers
  glad/, GLFW/, KHR/      vendored GLAD and GLFW headers
scripts/
  build_box2d.bat         compiles src/box2d into lib/libbox2d.a
```

`gfx::drawBox(x, y, width, height, angleRadians, r, g, b)` draws a filled
rectangle. `x` and `y` are the center in pixels, and a positive angle turns the
box clockwise on screen.

Everything in `core/` lives in the `gfx` namespace and uses RAII: an object
acquires its resource in its constructor and releases it in its destructor.
Startup failures throw `std::runtime_error`, which `main` catches and prints.

## Troubleshooting

- **`g++` or `ar` is not recognized, or the build fails with no error text.**
  The MinGW `bin` folder is not on your `PATH`. Add it, then restart VS Code.
- **`cannot open output file main.exe: Permission denied`.** The program is
  still running. Close its window and build again.
- **Linker errors mentioning `box2d` or `libstdc++`.** `lib/libbox2d.a` came
  from a different compiler. Delete it or run `Build Box2D library`.
- **Program compiles but won't run.** `glfw3.dll` must sit next to `main.exe`.
