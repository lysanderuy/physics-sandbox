# Physics Sandbox

A 2D physics demo in C++ and OpenGL, built on Box2D. Four boxes with different
materials slide, tumble and settle on a static ground, and holding the left
mouse button spawns more. `src/main.cpp` is built up one checkpoint at a time,
so each commit in the history adds one concept.

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

- **Spawn:** hold the left mouse button. A box appears at the cursor, at most
  one every quarter second.
- `Esc` closes the window.

## How it works

The window is 800x600 pixels with the origin at the top left and y pointing
down. Box2D works in meters with y pointing up, and 32 pixels make one meter
(`kPixelsPerMeter`). The world origin is the bottom left corner of the window.

The simulation runs at a fixed 60 Hz, driven by a time accumulator. A frame
longer than `kMaxFrameTime` is clamped, so a stall can never trigger a long
burst of steps.

Box2D positions are body centers and `SetAsBox` takes half extents.
`drawBoxBody` converts both to pixels and negates the angle, because rotation
counter-clockwise in Box2D is clockwise on a y-down screen.

Each box has its own density, friction and restitution. Bodies that fall below
`kRemoveBelowY` meters are destroyed.

The values to tune are the constants at the top of `src/main.cpp`:
`kGravity`, `kPixelsPerMeter`, `kStepDt`, `kSpawnCooldown` and `kSpawnHalfSize`.

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
  main.cpp                world, bodies, spawning, main loop
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
