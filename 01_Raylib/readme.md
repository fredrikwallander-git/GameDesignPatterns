# Raylib Setup Guide

## What is Raylib?

**Raylib** is a simple and easy-to-use game programming library written in C. It's designed to be beginner-friendly while still being powerful enough for real games.

**Why Raylib for learning design patterns?**
- **Simple API** - No complex setup, just include and go
- **Cross-platform** - Works on Windows, Mac, Linux
- **Pure C** - Works perfectly with C++
- **Lightweight** - No heavy engine overhead
- **Great for prototyping** - Quick to test ideas and patterns

**What can you make?**
- 2D games (platformers, top-down, puzzle games)
- 3D games (first-person, third-person)
- Visualizations and simulations
- Game prototypes to test design patterns

---

## Installation

### Windows (Visual Studio 2022/2026)

**Step 1: Download Raylib**
1. Go to https://github.com/raysan5/raylib/releases/latest
2. Download **`raylib-5.5_win64_msvc16.zip`** (or latest win64 MSVC version)
    - **Important:** Make sure it says **win64**, not win32!
3. Extract the ZIP file
4. Copy the `include` and `lib` folders to `C:\raylib\`

You should now have:
```
C:\raylib\
├── include\
│   ├── raylib.h
│   ├── rayconfig.h
│   └── ...
└── lib\
    ├── raylib.lib
    └── raylib.dll
```

**Step 2: Create a new Visual Studio project**
1. Open Visual Studio
2. Create New Project → **Empty Project (C++)**
3. Name it `RaylibGame`
4. Make sure platform is set to **x64** (not x86)

**Step 3: Add a source file**
1. Right-click **Source Files** in Solution Explorer
2. Add → New Item → **C++ File (.cpp)**
3. Name it `main.cpp`

**Step 4: Configure project settings**

Right-click on your project → **Properties**

**Make sure:** Configuration: **All Configurations**, Platform: **Active(x64)**

**A. Set Include Directories:**
1. Expand **VC++ Directories** in left sidebar
2. Click **Include Directories**
3. Click the dropdown arrow → Edit
4. Add new line: `C:\raylib\include`

**B. Set Library Directories:**
1. Still in **VC++ Directories**
2. Click **Library Directories**
3. Click the dropdown arrow → Edit
4. Add new line: `C:\raylib\lib`

**C. Set Additional Include Directories:**
1. Expand **C/C++**
2. Click **Additional Include Directories**
3. Click the dropdown arrow → Edit
4. Add new line: `C:\raylib\include`

**D. Link the Libraries:**
1. Expand **Linker** in left sidebar
2. Click **Input**
3. Click **Additional Dependencies**
4. Click the dropdown arrow → Edit
5. At the **very top**, add these to the start:
```
raylib.lib;winmm.lib;gdi32.lib;opengl32.lib;
```

**D. Click Apply, then OK** to close Properties.

**Step 5: Copy raylib.dll to output folder**
1. Copy `raylib.dll` from `C:\raylib\lib`
2. Paste it into your project folder (same folder as your `.vcxproj` file)
3. **OR** paste it into `x64\Debug\` after you build once

**You're ready to code!**

---

### macOS (Xcode or VS Code)

**Step 1: Install Raylib via Homebrew**
```bash
brew install raylib
```

**Step 2: Create a C++ file**
```bash
mkdir RaylibGame
cd RaylibGame
touch main.cpp
```

**Step 3: Compile and run**
```bash
g++ main.cpp -o game -lraylib -framework OpenGL -framework Cocoa -framework IOKit
./game
```

Or use VS Code with a configured `tasks.json` for building.

---

## Your First Raylib Program

Edit your file called `main.cpp`:

```cpp
#include "raylib.h"

int main()
{
    // Initialize window
    const int screenWidth = 800;
    const int screenHeight = 600;
    
    InitWindow(screenWidth, screenHeight, "My First Raylib Game");
    SetTargetFPS(60);
    
    // Game loop
    while (!WindowShouldClose())
    {
        // Update
        // (nothing yet)
        
        // Draw
        BeginDrawing();
        
        ClearBackground(RAYWHITE);
        DrawText("Hello, Raylib!", 300, 250, 30, DARKGRAY);
        
        EndDrawing();
    }
    
    // Cleanup
    CloseWindow();
    
    return 0;
}
```

**Build and run!** You should see a window with "Hello, Raylib!" text.

---

## Understanding the Structure

Every Raylib game follows this pattern:

```cpp
// 1. INITIALIZATION
InitWindow(width, height, "Title");
SetTargetFPS(60);

// Load resources here (textures, sounds, etc.)

// 2. GAME LOOP
while (!WindowShouldClose())
{
    // 2a. UPDATE - Game logic, input, physics
    
    // 2b. DRAW - Render everything
    BeginDrawing();
    ClearBackground(RAYWHITE);
    
    // Draw calls here
    
    EndDrawing();
}

// 3. CLEANUP
// Unload resources
CloseWindow();
```

**Key points:**
- **Update** section: Handle input, update positions, game logic
- **Draw** section: Only drawing, no logic!
- Everything between `BeginDrawing()` and `EndDrawing()` is rendered to screen
- `ClearBackground()` wipes the screen each frame

---

## Adding a Moveable Player

Let's create a simple square that moves with WASD:

```cpp
#include "raylib.h"

int main()
{
    // Window setup
    const int screenWidth = 800;
    const int screenHeight = 600;
    
    InitWindow(screenWidth, screenHeight, "WASD Movement");
    SetTargetFPS(60);
    
    // Player setup
    Vector2 playerPosition = { 400, 300 };  // Center of screen
    float playerSize = 50.0f;
    float playerSpeed = 200.0f;  // pixels per second
    
    // Game loop
    while (!WindowShouldClose())
    {
        // ──────── UPDATE ────────
        float deltaTime = GetFrameTime();  // Time since last frame
        
        // Input handling
        if (IsKeyDown(KEY_W)) playerPosition.y -= playerSpeed * deltaTime;
        if (IsKeyDown(KEY_S)) playerPosition.y += playerSpeed * deltaTime;
        if (IsKeyDown(KEY_A)) playerPosition.x -= playerSpeed * deltaTime;
        if (IsKeyDown(KEY_D)) playerPosition.x += playerSpeed * deltaTime;
        
        // Keep player on screen
        if (playerPosition.x < 0) playerPosition.x = 0;
        if (playerPosition.x > screenWidth - playerSize) playerPosition.x = screenWidth - playerSize;
        if (playerPosition.y < 0) playerPosition.y = 0;
        if (playerPosition.y > screenHeight - playerSize) playerPosition.y = screenHeight - playerSize;
        
        // ──────── DRAW ────────
        BeginDrawing();
        
        ClearBackground(RAYWHITE);
        
        // Draw player as a rectangle
        DrawRectangleV(playerPosition, { playerSize, playerSize }, BLUE);
        
        // Draw instructions
        DrawText("Use WASD to move", 10, 10, 20, DARKGRAY);
        
        EndDrawing();
    }
    
    CloseWindow();
    
    return 0;
}
```

**Try it!** You should be able to move the blue square around with WASD.

---

## Key Raylib Concepts Used

### Delta Time
```cpp
float deltaTime = GetFrameTime();
```
Time (in seconds) since the last frame. **Always use this for movement!**

**Why?**
- Makes movement framerate-independent
- `speed * deltaTime` = consistent movement on any computer

### Input Functions
```cpp
IsKeyDown(KEY_W)      // True while key is held
IsKeyPressed(KEY_W)   // True only on first frame of press
IsKeyReleased(KEY_W)  // True only on frame key is released
```

### Drawing Functions
```cpp
DrawRectangle(x, y, width, height, color)
DrawRectangleV(position, size, color)  // Vector version
DrawCircle(x, y, radius, color)
DrawText(text, x, y, fontSize, color)
DrawTexture(texture, x, y, color)
```

### Raylib Vector2
```cpp
Vector2 pos = { 100, 200 };  // x, y
pos.x = 150;
pos.y = 250;
```

---

## Common Raylib Functions You'll Use

| Category | Function | Purpose |
|----------|----------|---------|
| **Window** | `InitWindow()` | Create window |
| | `CloseWindow()` | Cleanup |
| | `WindowShouldClose()` | Check if user closed window |
| | `SetTargetFPS()` | Lock framerate |
| **Time** | `GetFrameTime()` | Delta time in seconds |
| | `GetTime()` | Time since init |
| **Input** | `IsKeyDown(KEY)` | Check if key held |
| | `IsKeyPressed(KEY)` | Check if key just pressed |
| | `IsMouseButtonPressed(MOUSE_BUTTON)` | Mouse click |
| | `GetMousePosition()` | Mouse x, y |
| **Drawing** | `BeginDrawing()` | Start frame |
| | `EndDrawing()` | Finish frame |
| | `ClearBackground(color)` | Fill screen with color |
| | `DrawRectangle()` | Draw rectangle |
| | `DrawCircle()` | Draw circle |
| | `DrawText()` | Draw text |
| **Colors** | `RED`, `BLUE`, `GREEN`, `YELLOW`, `BLACK`, `WHITE`, `RAYWHITE`, `DARKGRAY` | Predefined colors |

---

## Project Structure for Design Patterns

As we add design patterns, you might organize your code like this:

```
RaylibGame/
├── main.cpp              // Entry point, game loop
├── Player.h              // Player class
├── Player.cpp
├── Enemy.h               // Enemy class
├── Enemy.cpp
├── ObjectPool.h          // Object Pool pattern
├── ObjectPool.cpp
├── GameManager.h         // Singleton pattern
├── GameManager.cpp
└── raylib.dll            // (Windows only)
```

**Tip:** Keep `main.cpp` simple - just initialization, game loop, and cleanup. Move game logic into classes.

---

## Next Steps

Now that you have a moving player, you can:

1. **Add more objects** - enemies, collectibles, obstacles
2. **Implement design patterns** - use patterns to manage these objects
3. **Add collisions** - `CheckCollisionRecs()` for rectangles
4. **Add sprites** - `LoadTexture()` and `DrawTexture()`
5. **Add sounds** - `LoadSound()` and `PlaySound()`

---

## Troubleshooting

### "Cannot open include file 'raylib.h'"
- Check your include directories in project settings
- Make sure the path points to the folder **containing** `raylib.h`

### "Unresolved external symbol"
- Check your library directories in linker settings
- Make sure you added all the required libraries to Additional Dependencies
- Check that you're building for x64 (not x86)

### Window opens and immediately closes
- Make sure you have the `while (!WindowShouldClose())` loop
- Check you're not calling `CloseWindow()` too early

### "raylib.dll not found"
- Copy `raylib.dll` to the same folder as your .exe
- Or add the raylib `lib` folder to your system PATH

---

## Summary

**Raylib gives you:**
- ✅ Simple window and input handling
- ✅ Easy drawing functions
- ✅ Cross-platform support
- ✅ Perfect testbed for design patterns

**You learned:**
- How to set up Raylib in your IDE
- The basic game loop structure
- How to handle input with WASD
- How to use delta time for smooth movement

**Next up:** We'll start implementing design patterns to turn this into a real game!