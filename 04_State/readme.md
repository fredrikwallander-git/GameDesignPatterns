# State Pattern

## The Problem

Your **Zombie Horde** game is growing. Now you have:
- Main menu
- Playing the game
- Paused
- Game over screen

Your `main.cpp` looks like this disaster:

```cpp
enum GamePhase { MENU, PLAYING, PAUSED, GAME_OVER };
GamePhase currentPhase = MENU;

while (!WindowShouldClose())
{
    // Update
    if (currentPhase == MENU)
    {
        if (IsKeyPressed(KEY_SPACE))
            currentPhase = PLAYING;
    }
    else if (currentPhase == PLAYING)
    {
        if (IsKeyPressed(KEY_P))
            currentPhase = PAUSED;
        
        if (GameManager::Instance().IsGameOver())
            currentPhase = GAME_OVER;
        
        player.Update(deltaTime);
        bulletPool.UpdateAll(deltaTime);
        zombiePool.UpdateAll(deltaTime);
        // ... 50 more lines
    }
    else if (currentPhase == PAUSED)
    {
        if (IsKeyPressed(KEY_P))
            currentPhase = PLAYING;
        
        if (IsKeyPressed(KEY_ESCAPE))
            currentPhase = MENU;
    }
    else if (currentPhase == GAME_OVER)
    {
        if (IsKeyPressed(KEY_R))
            currentPhase = MENU;
    }
    
    // Draw
    if (currentPhase == MENU)
    {
        DrawText("ZOMBIE HORDE", 200, 200, 60, RED);
        DrawText("Press SPACE to Start", 250, 300, 20, WHITE);
    }
    else if (currentPhase == PLAYING)
    {
        player.Draw();
        bulletPool.DrawAll();
        zombiePool.DrawAll();
        // ... 30 more lines
    }
    else if (currentPhase == PAUSED)
    {
        // Draw everything + pause overlay
        player.Draw();
        bulletPool.DrawAll();
        zombiePool.DrawAll();
        DrawRectangle(0, 0, 800, 600, Fade(BLACK, 0.5f));
        DrawText("PAUSED", 300, 250, 40, WHITE);
    }
    else if (currentPhase == GAME_OVER)
    {
        DrawText("GAME OVER", 280, 200, 50, RED);
        DrawText(TextFormat("Score: %d", score), 320, 300, 30, WHITE);
    }
}
```

### What's Wrong?

**Problem 1: Giant if/else chain**
- Hard to read
- Hard to maintain
- Easy to make mistakes

**Problem 2: No encapsulation**
```cpp
// Menu logic mixed with playing logic mixed with pause logic
// All in one giant function!
```

**Problem 3: Adding new states is painful**
```cpp
// Want to add SETTINGS state?
// Now you need to modify:
// - The enum
// - Update section (add another if/else)
// - Draw section (add another if/else)
// - Every state transition (check all the code)
```

**Problem 4: State transition bugs**
```cpp
if (IsKeyPressed(KEY_ESCAPE))
{
    currentPhase = MENU;
    // Oops! Forgot to reset the game
    // Forgot to stop sounds
    // Forgot to clear bullets
}
```

---

## The Solution: State Pattern

**Each game state is its own class** with clear responsibilities.

Think of it like **TV channels**:
- Each channel (state) has its own content
- Switching channels is clean and simple
- Each channel doesn't know about others
- Remote control (state manager) handles transitions

### Key Characteristics:
1. **Each state is a class** (MenuState, PlayingState, etc.)
2. **Common interface** (Enter, Exit, Update, Draw)
3. **State manager** handles transitions
4. **Clean separation** - states don't know about each other

---

## Implementation in Zombie Horde

### GameState.h (Base Class)

```cpp
#ifndef GAME_STATE_H
#define GAME_STATE_H

#include "raylib.h"

// Base class for all game states
class GameState
{
public:
    virtual ~GameState() = default;
    
    // Called when entering this state
    virtual void Enter() = 0;
    
    // Called when exiting this state
    virtual void Exit() = 0;
    
    // Update game logic (returns name of next state, or empty string to stay)
    virtual const char* Update(float deltaTime) = 0;
    
    // Draw everything
    virtual void Draw() const = 0;
    
    // Get state name (for debugging)
    virtual const char* GetName() const = 0;
};

#endif
```

---

### MenuState.h

```cpp
#ifndef MENU_STATE_H
#define MENU_STATE_H

#include "GameState.h"

class MenuState : public GameState
{
private:
    float titleAlpha;  // For fade-in effect
    
public:
    void Enter() override;
    void Exit() override;
    const char* Update(float deltaTime) override;
    void Draw() const override;
    const char* GetName() const override { return "Menu"; }
};

#endif
```

### MenuState.cpp

```cpp
#include "MenuState.h"
#include "GameManager.h"

void MenuState::Enter()
{
    // Reset game when entering menu
    GameManager::Instance().ResetGame();
    titleAlpha = 0.0f;
}

void MenuState::Exit()
{
    // Cleanup if needed
}

const char* MenuState::Update(float deltaTime)
{
    // Fade in title
    titleAlpha += deltaTime * 2.0f;
    if (titleAlpha > 1.0f) titleAlpha = 1.0f;
    
    // Check for state transitions
    if (IsKeyPressed(KEY_SPACE))
    {
        return "Playing";  // Request transition to Playing state
    }
    
    if (IsKeyPressed(KEY_ESCAPE))
    {
        return "Exit";  // Request to quit game
    }
    
    return "";  // Stay in menu
}

void MenuState::Draw() const
{
    ClearBackground(DARKGRAY);
    
    // Title with fade-in
    Color titleColor = Fade(RED, titleAlpha);
    DrawText("ZOMBIE HORDE", 200, 200, 60, titleColor);
    
    // Instructions
    DrawText("Press SPACE to Start", 250, 300, 20, WHITE);
    DrawText("Press ESC to Quit", 270, 330, 20, GRAY);
    
    // Version or credits
    DrawText("v1.0", 10, 570, 20, DARKGRAY);
}
```

---

### PlayingState.h

```cpp
#ifndef PLAYING_STATE_H
#define PLAYING_STATE_H

#include "GameState.h"
#include "Player.h"
#include "BulletPool.h"
#include "ZombiePool.h"

class PlayingState : public GameState
{
private:
    Player player;
    BulletPool bulletPool;
    ZombiePool zombiePool;
    
    float spawnTimer;
    float spawnInterval;
    
public:
    PlayingState();
    
    void Enter() override;
    void Exit() override;
    const char* Update(float deltaTime) override;
    void Draw() const override;
    const char* GetName() const override { return "Playing"; }
};

#endif
```

### PlayingState.cpp

```cpp
#include "PlayingState.h"
#include "GameManager.h"

PlayingState::PlayingState()
    : bulletPool(100)
    , zombiePool(50)
    , spawnTimer(0.0f)
    , spawnInterval(2.0f)
{
}

void PlayingState::Enter()
{
    // Reset player position
    player.Reset();
    
    // Clear any leftover bullets/zombies
    spawnTimer = 0.0f;
}

void PlayingState::Exit()
{
    // Could save score, stats, etc.
}

const char* PlayingState::Update(float deltaTime)
{
    GameManager& game = GameManager::Instance();
    
    // Check for pause
    if (IsKeyPressed(KEY_P) || IsKeyPressed(KEY_ESCAPE))
    {
        return "Paused";
    }
    
    // Check for game over
    if (game.IsGameOver())
    {
        return "GameOver";
    }
    
    // Update player
    player.Update(deltaTime);
    
    // Player shooting
    if (IsKeyPressed(KEY_SPACE))
    {
        Vector2 direction = {1, 0};  // Simplified: shoot right
        bulletPool.Spawn(player.GetPosition(), direction, 500.0f);
    }
    
    // Update bullets
    bulletPool.UpdateAll(deltaTime);
    
    // Update zombies
    zombiePool.UpdateAll(deltaTime);
    
    // Spawn zombies
    spawnTimer += deltaTime;
    if (spawnTimer >= spawnInterval)
    {
        spawnTimer = 0.0f;
        
        // Spawn zombie at random position
        Vector2 spawnPos = {
            (float)GetRandomValue(0, 800),
            (float)GetRandomValue(0, 600)
        };
        zombiePool.Spawn(spawnPos);
    }
    
    // Collision detection (simplified)
    // TODO: Check bullet-zombie collisions
    
    return "";  // Stay in playing state
}

void PlayingState::Draw() const
{
    ClearBackground(DARKGREEN);
    
    // Draw game objects
    player.Draw();
    bulletPool.DrawAll();
    zombiePool.DrawAll();
    
    // Draw UI
    GameManager& game = GameManager::Instance();
    DrawText(TextFormat("Score: %d", game.GetScore()), 10, 10, 20, WHITE);
    DrawText(TextFormat("Wave: %d", game.GetCurrentWave()), 10, 35, 20, YELLOW);
    DrawText(TextFormat("Health: %d", game.GetPlayerHealth()), 10, 60, 20, RED);
}
```

---

### PausedState.h

```cpp
#ifndef PAUSED_STATE_H
#define PAUSED_STATE_H

#include "GameState.h"

class PausedState : public GameState
{
private:
    // We need a reference to the playing state to draw it underneath
    const GameState* previousState;
    
public:
    void Enter() override;
    void Exit() override;
    const char* Update(float deltaTime) override;
    void Draw() const override;
    const char* GetName() const override { return "Paused"; }
    
    void SetPreviousState(const GameState* state) { previousState = state; }
};

#endif
```

### PausedState.cpp

```cpp
#include "PausedState.h"
#include "GameManager.h"

void PausedState::Enter()
{
    GameManager::Instance().SetPaused(true);
}

void PausedState::Exit()
{
    GameManager::Instance().SetPaused(false);
}

const char* PausedState::Update(float deltaTime)
{
    // Resume
    if (IsKeyPressed(KEY_P) || IsKeyPressed(KEY_ESCAPE))
    {
        return "Playing";
    }
    
    // Quit to menu
    if (IsKeyPressed(KEY_Q))
    {
        return "Menu";
    }
    
    return "";  // Stay paused
}

void PausedState::Draw() const
{
    // Draw the game underneath (frozen)
    if (previousState)
    {
        previousState->Draw();
    }
    
    // Draw dark overlay
    DrawRectangle(0, 0, 800, 600, Fade(BLACK, 0.6f));
    
    // Draw pause text
    DrawText("PAUSED", 300, 250, 50, WHITE);
    DrawText("Press P or ESC to Resume", 230, 320, 20, LIGHTGRAY);
    DrawText("Press Q to Quit to Menu", 240, 350, 20, LIGHTGRAY);
}
```

---

### GameOverState.h

```cpp
#ifndef GAME_OVER_STATE_H
#define GAME_OVER_STATE_H

#include "GameState.h"

class GameOverState : public GameState
{
private:
    int finalScore;
    int finalWave;
    float displayTimer;
    
public:
    void Enter() override;
    void Exit() override;
    const char* Update(float deltaTime) override;
    void Draw() const override;
    const char* GetName() const override { return "GameOver"; }
};

#endif
```

### GameOverState.cpp

```cpp
#include "GameOverState.h"
#include "GameManager.h"

void GameOverState::Enter()
{
    // Capture final stats
    GameManager& game = GameManager::Instance();
    finalScore = game.GetScore();
    finalWave = game.GetCurrentWave();
    displayTimer = 0.0f;
}

void GameOverState::Exit()
{
    // Cleanup
}

const char* GameOverState::Update(float deltaTime)
{
    displayTimer += deltaTime;
    
    // Wait 1 second before allowing restart
    if (displayTimer > 1.0f)
    {
        if (IsKeyPressed(KEY_R) || IsKeyPressed(KEY_SPACE))
        {
            return "Menu";
        }
    }
    
    return "";  // Stay in game over
}

void GameOverState::Draw() const
{
    ClearBackground(MAROON);
    
    // Game over text
    DrawText("GAME OVER", 260, 150, 60, RED);
    
    // Stats
    DrawText(TextFormat("Final Score: %d", finalScore), 280, 250, 30, WHITE);
    DrawText(TextFormat("Wave Reached: %d", finalWave), 270, 290, 30, YELLOW);
    
    // Restart prompt (fade in after delay)
    if (displayTimer > 1.0f)
    {
        float alpha = (displayTimer - 1.0f) * 2.0f;
        if (alpha > 1.0f) alpha = 1.0f;
        
        Color promptColor = Fade(WHITE, alpha);
        DrawText("Press R or SPACE to Continue", 200, 400, 25, promptColor);
    }
}
```

---

### StateManager.h

```cpp
#ifndef STATE_MANAGER_H
#define STATE_MANAGER_H

#include "GameState.h"
#include <map>
#include <string>

class StateManager
{
private:
    std::map<std::string, GameState*> states;
    GameState* currentState;
    GameState* nextState;
    
public:
    StateManager();
    ~StateManager();
    
    // Register a state
    void AddState(const std::string& name, GameState* state);
    
    // Change to a different state
    void ChangeState(const std::string& name);
    
    // Update current state
    void Update(float deltaTime);
    
    // Draw current state
    void Draw() const;
    
    // Get current state name
    const char* GetCurrentStateName() const;
};

#endif
```

### StateManager.cpp

```cpp
#include "StateManager.h"
#include <iostream>

StateManager::StateManager()
    : currentState(nullptr)
    , nextState(nullptr)
{
}

StateManager::~StateManager()
{
    // Clean up all states
    for (auto& pair : states)
    {
        delete pair.second;
    }
}

void StateManager::AddState(const std::string& name, GameState* state)
{
    states[name] = state;
}

void StateManager::ChangeState(const std::string& name)
{
    auto it = states.find(name);
    if (it != states.end())
    {
        nextState = it->second;
    }
    else
    {
        std::cout << "ERROR: State '" << name << "' not found!" << std::endl;
    }
}

void StateManager::Update(float deltaTime)
{
    // Handle state transition
    if (nextState != nullptr && nextState != currentState)
    {
        // Exit old state
        if (currentState != nullptr)
        {
            std::cout << "Exiting state: " << currentState->GetName() << std::endl;
            currentState->Exit();
        }
        
        // Enter new state
        currentState = nextState;
        nextState = nullptr;
        std::cout << "Entering state: " << currentState->GetName() << std::endl;
        currentState->Enter();
    }
    
    // Update current state
    if (currentState != nullptr)
    {
        const char* requestedState = currentState->Update(deltaTime);
        
        // Check if state wants to transition
        if (requestedState != nullptr && requestedState[0] != '\0')
        {
            ChangeState(requestedState);
        }
    }
}

void StateManager::Draw() const
{
    if (currentState != nullptr)
    {
        currentState->Draw();
    }
}

const char* StateManager::GetCurrentStateName() const
{
    return currentState ? currentState->GetName() : "None";
}
```

---

## How to Use It

### main.cpp (Clean!)

```cpp
#include "raylib.h"
#include "StateManager.h"
#include "MenuState.h"
#include "PlayingState.h"
#include "PausedState.h"
#include "GameOverState.h"

int main()
{
    InitWindow(800, 600, "Zombie Horde");
    SetTargetFPS(60);
    
    // Create state manager
    StateManager stateManager;
    
    // Register all states
    stateManager.AddState("Menu", new MenuState());
    stateManager.AddState("Playing", new PlayingState());
    stateManager.AddState("Paused", new PausedState());
    stateManager.AddState("GameOver", new GameOverState());
    
    // Start in menu
    stateManager.ChangeState("Menu");
    
    // Game loop - MUCH CLEANER!
    while (!WindowShouldClose())
    {
        float deltaTime = GetFrameTime();
        
        // Update current state
        stateManager.Update(deltaTime);
        
        // Draw current state
        BeginDrawing();
        stateManager.Draw();
        
        // Debug: Show current state
        DrawText(stateManager.GetCurrentStateName(), 700, 10, 20, YELLOW);
        
        EndDrawing();
    }
    
    CloseWindow();
    return 0;
}
```

**Look how clean that is!** All the complexity is hidden in the states.

---

## How It Works

### State Lifecycle

```
1. ChangeState("Playing") is called
   ↓
2. StateManager stores nextState = PlayingState
   ↓
3. On next Update():
   - currentState->Exit() is called (MenuState cleanup)
   - currentState = nextState
   - currentState->Enter() is called (PlayingState setup)
   ↓
4. Every frame:
   - currentState->Update() runs
   - Returns name of next state or ""
   ↓
5. If Update() returns "Paused":
   - Cycle repeats from step 1
```

### State Transition Diagram

```
    ┌──────┐  SPACE   ┌─────────┐   P/ESC   ┌────────┐
    │ Menu │─────────→│ Playing │←─────────→│ Paused │
    └──────┘          └─────────┘           └────────┘
       ↑                   │                     │
       │                   │ Game Over           │ Q
       │                   ↓                     │
       │              ┌──────────┐              │
       └──────────────│ GameOver │←─────────────┘
          R/SPACE     └──────────┘
```

---

## Exercise 1: Basic Extension - Settings State

**Goal:** Add a Settings state accessible from the menu.

**Requirements:**
1. Create `SettingsState` class
2. Accessible from Menu with 'S' key
3. Display 3 options:
    - Master Volume (0-100)
    - Difficulty (Easy/Normal/Hard)
    - Show FPS (On/Off)
4. Use arrow keys to navigate
5. Use ENTER to change values
6. Press ESC to return to Menu
7. Save settings to GameManager

**Starter code:**
```cpp
class SettingsState : public GameState
{
private:
    int selectedOption;  // 0, 1, or 2
    
public:
    void Enter() override;
    void Exit() override;
    const char* Update(float deltaTime) override;
    void Draw() const override;
    const char* GetName() const override { return "Settings"; }
};
```

**Expected flow:**
```
Menu → (Press S) → Settings → (Press ESC) → Menu
```

---

## Exercise 2: Intermediate Challenge - State Transitions with Fade

**Goal:** Add fade-in/fade-out transitions between states.

**Requirements:**
1. Create `FadeTransition` class
2. When changing states:
    - Fade out current state (0.5 seconds)
    - Change state
    - Fade in new state (0.5 seconds)
3. Draw black overlay with changing alpha
4. States should NOT update during transitions

**Add to StateManager:**
```cpp
private:
    FadeTransition transition;
    bool inTransition;
    
public:
    void Update(float deltaTime)
    {
        if (inTransition)
        {
            transition.Update(deltaTime);
            if (transition.IsComplete())
            {
                // Actually change state
                inTransition = false;
            }
        }
        else
        {
            // Normal update
        }
    }
```

**Test it:** All state changes should now smoothly fade.

---

## Exercise 3: Advanced - State History (Back Button)

**Goal:** Implement a "back" system that returns to previous state.

**Requirements:**
1. Keep a stack of previous states
2. Add `GoBack()` method to StateManager
3. States can return "Back" instead of a state name
4. Example flow:
   ```
   Menu → Playing → Paused → (Go Back) → Playing → (Go Back) → Menu
   ```
5. Limit history to 10 states (prevent stack overflow)

**Challenge:** Should PlayingState be in the stack when you go to PausedState? What happens if you pause, go back, and the game is over?

---

## Exercise 4: Advanced (Optional) - State Serialization

**Goal:** Save/load game state to resume later.

**Requirements:**
1. Add `Serialize()` and `Deserialize()` to GameState
2. Save:
    - Current state name
    - Player position, health
    - All active bullets positions/velocities
    - All active zombies
    - Score, wave, etc.
3. Load should restore exact game state
4. Add "Save & Quit" option in PausedState
5. Add "Continue" option in Menu (if save exists)

**Hint:** Use a file format like JSON or just plain text.

---

## Common Mistakes

### ❌ Mistake 1: Forgetting to call Enter/Exit

```cpp
// WRONG - directly changing state
currentState = menuState;  // MenuState.Enter() never called!
```

**Fix:** Always use ChangeState
```cpp
// CORRECT
stateManager.ChangeState("Menu");  // Properly calls Exit/Enter
```

---

### ❌ Mistake 2: States knowing about each other

```cpp
// WRONG - PlayingState directly referencing MenuState
class PlayingState
{
    MenuState* menuState;  // BAD!
    
    void GoToMenu()
    {
        menuState->Enter();  // States shouldn't know about each other!
    }
};
```

**Fix:** Return state name, let StateManager handle it
```cpp
// CORRECT
const char* PlayingState::Update(float deltaTime)
{
    if (shouldGoToMenu)
    {
        return "Menu";  // Just return the name
    }
    return "";
}
```

---

### ❌ Mistake 3: Not cleaning up in Exit

```cpp
void PlayingState::Exit()
{
    // Empty! Forgot to clean up!
}

// Later:
// Bullets still flying, zombies still active, sounds still playing!
```

**Fix:** Clean up everything
```cpp
void PlayingState::Exit()
{
    bulletPool.DeactivateAll();
    zombiePool.DeactivateAll();
    StopAllSounds();
    // etc.
}
```

---

### ❌ Mistake 4: Memory leaks with state pointers

```cpp
// WRONG - creating new states every time
void SwitchToPlaying()
{
    currentState = new PlayingState();  // Memory leak!
}
```

**Fix:** Create states once, reuse them
```cpp
// CORRECT - states created once in main
StateManager stateManager;
stateManager.AddState("Playing", new PlayingState());  // Created once
stateManager.ChangeState("Playing");  // Reused
```

---

### ❌ Mistake 5: Update and Draw logic mixed

```cpp
void PlayingState::Update(float deltaTime)
{
    player.Update(deltaTime);
    DrawText("Score: ...", 10, 10, 20, WHITE);  // WRONG! Drawing in Update!
}
```

**Fix:** Keep Update and Draw separate
```cpp
void PlayingState::Update(float deltaTime)
{
    player.Update(deltaTime);  // Logic only
}

void PlayingState::Draw() const
{
    player.Draw();  // Drawing only
    DrawText("Score: ...", 10, 10, 20, WHITE);
}
```

---

## When to Use State Pattern

### ✅ Good Use Cases:
- **Game states** (Menu, Playing, Paused, GameOver)
- **AI states** (Idle, Patrol, Chase, Attack, Flee)
- **UI screens** (Login, Dashboard, Settings, Profile)
- **Character states** (Standing, Walking, Jumping, Attacking)
- **Connection states** (Disconnected, Connecting, Connected, Error)

### ❌ Bad Use Cases:
- **Simple boolean flags** (if there's only 2 states, use a bool)
- **One-time events** (don't need a state for a button click)
- **Constant transitions** (if changing every frame, use something else)

### The Rule of Thumb:
**"If you have 3+ states with different behaviors and clear transitions, use State Pattern."**

---

## State Pattern vs Other Approaches

### vs if/else Chain
```cpp
// Without State Pattern
if (phase == MENU) { ... }
else if (phase == PLAYING) { ... }
else if (phase == PAUSED) { ... }
// 500 lines of spaghetti!

// With State Pattern
currentState->Update(deltaTime);  // Clean!
```

### vs Switch Statement
```cpp
// Without State Pattern
switch (phase)
{
    case MENU: /* 50 lines */ break;
    case PLAYING: /* 200 lines */ break;
    case PAUSED: /* 30 lines */ break;
}

// With State Pattern
// Each case is its own class - much cleaner!
```

---

## Summary

✅ **State Pattern** encapsulates each state in its own class
✅ **Common interface** with Enter, Exit, Update, Draw
✅ **StateManager** handles transitions
✅ **Cleaner code** - no giant if/else chains
✅ **Easy to extend** - just add a new state class

**The pattern you just learned:**
```cpp
// Base
class GameState
{
    virtual void Enter() = 0;
    virtual void Exit() = 0;
    virtual const char* Update(float dt) = 0;
    virtual void Draw() const = 0;
};

// Concrete
class MenuState : public GameState { /* ... */ };

// Manager
class StateManager
{
    void ChangeState(const std::string& name);
    void Update(float dt);
    void Draw() const;
};
```

**Before State Pattern:**
```cpp
main.cpp: 800 lines of if/else spaghetti
```

**After State Pattern:**
```cpp
main.cpp: 30 lines (clean!)
MenuState.cpp: 50 lines (focused)
PlayingState.cpp: 100 lines (focused)
PausedState.cpp: 40 lines (focused)
GameOverState.cpp: 50 lines (focused)
```

**Next Pattern:** Factory Pattern - creating different zombie types cleanly!