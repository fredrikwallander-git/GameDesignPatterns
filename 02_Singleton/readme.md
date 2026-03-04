# Singleton Pattern

## The Problem

You're building **Zombie Horde** and you quickly run into these issues:

**Problem 1: Screen dimensions everywhere**
```cpp
// In Player.cpp
if (position.x > 800) position.x = 0;  // Hard-coded!

// In Zombie.cpp
if (position.x > 800) position.x = 0;  // Same hard-coded value!

// In Bullet.cpp
if (position.y < 0 || position.y > 600) return;  // More hard-coding!
```

Every class needs to know screen size, but passing it to every constructor is messy.

**Problem 2: Score tracking**
```cpp
// How does the UI access the score?
// How do zombies increment it when they die?
// Do we pass a score pointer to every zombie?
```

**Problem 3: Global game state**
```cpp
// Is the game paused?
// What wave are we on?
// Is debug mode enabled?
```

You need **one central place** to store and access game-wide data.

---

## The Solution: Singleton Pattern

**Singleton** ensures a class has **only one instance** and provides a **global access point** to it.

Think of it like a **government** - there's only one, and everyone can access it.

### Key Characteristics:
1. **One instance only** - can't create multiple GameManagers
2. **Global access** - any code can get to it with `GameManager::Instance()`
3. **Lazy initialization** - created when first needed

---

## Implementation in Zombie Horde

### GameManager.h

```cpp
#ifndef GAME_MANAGER_H
#define GAME_MANAGER_H

#include "raylib.h"

class GameManager
{
private:
    // Private constructor - can't create instances from outside
    GameManager();
    
    // Delete copy constructor and assignment operator
    // This prevents making copies of the singleton
    GameManager(const GameManager&) = delete;
    GameManager& operator=(const GameManager&) = delete;
    
    // Game state
    int score;
    int currentWave;
    int playerHealth;
    bool isPaused;
    bool isGameOver;
    
public:
    // The ONLY way to get the instance
    static GameManager& Instance()
    {
        static GameManager instance;  // Created once, lives forever
        return instance;
    }
    
    // Constants
    static const int SCREEN_WIDTH = 800;
    static const int SCREEN_HEIGHT = 600;
    static const int PLAYER_MAX_HEALTH = 100;
    
    // Score management
    void AddScore(int points);
    int GetScore() const { return score; }
    void ResetScore();
    
    // Wave management
    void NextWave();
    int GetCurrentWave() const { return currentWave; }
    
    // Health management
    void DamagePlayer(int damage);
    void HealPlayer(int amount);
    int GetPlayerHealth() const { return playerHealth; }
    bool IsPlayerDead() const { return playerHealth <= 0; }
    
    // Game state
    void SetPaused(bool paused) { isPaused = paused; }
    bool IsPaused() const { return isPaused; }
    void SetGameOver(bool gameOver) { isGameOver = gameOver; }
    bool IsGameOver() const { return isGameOver; }
    
    // Game control
    void ResetGame();
};

#endif
```

### GameManager.cpp

```cpp
#include "GameManager.h"
#include <algorithm>

// Private constructor - initialize default values
GameManager::GameManager()
    : score(0)
    , currentWave(1)
    , playerHealth(PLAYER_MAX_HEALTH)
    , isPaused(false)
    , isGameOver(false)
{
    // Could load settings from file here
}

void GameManager::AddScore(int points)
{
    score += points;
}

void GameManager::ResetScore()
{
    score = 0;
}

void GameManager::NextWave()
{
    currentWave++;
}

void GameManager::DamagePlayer(int damage)
{
    playerHealth -= damage;
    if (playerHealth < 0)
    {
        playerHealth = 0;
        isGameOver = true;
    }
}

void GameManager::HealPlayer(int amount)
{
    playerHealth += amount;
    if (playerHealth > PLAYER_MAX_HEALTH)
    {
        playerHealth = PLAYER_MAX_HEALTH;
    }
}

void GameManager::ResetGame()
{
    score = 0;
    currentWave = 1;
    playerHealth = PLAYER_MAX_HEALTH;
    isPaused = false;
    isGameOver = false;
}
```

---

## How to Use It

### Accessing the Singleton

```cpp
// Anywhere in your code:
GameManager& game = GameManager::Instance();

// Or use it directly:
GameManager::Instance().AddScore(10);
```

### Example: Zombie dies and adds score

```cpp
// In Zombie.cpp
void Zombie::TakeDamage(int damage)
{
    health -= damage;
    
    if (health <= 0)
    {
        // Add score when zombie dies
        GameManager::Instance().AddScore(points);
        isAlive = false;
    }
}
```

### Example: Player uses screen bounds

```cpp
// In Player.cpp
void Player::Update(float deltaTime)
{
    position.x += velocity.x * deltaTime;
    position.y += velocity.y * deltaTime;
    
    // Use singleton for screen dimensions
    if (position.x < 0) position.x = 0;
    if (position.x > GameManager::SCREEN_WIDTH) 
        position.x = GameManager::SCREEN_WIDTH;
    
    if (position.y < 0) position.y = 0;
    if (position.y > GameManager::SCREEN_HEIGHT) 
        position.y = GameManager::SCREEN_HEIGHT;
}
```

### Example: UI displays score

```cpp
// In main.cpp or UI.cpp
void DrawUI()
{
    GameManager& game = GameManager::Instance();
    
    // Draw score
    DrawText(TextFormat("Score: %d", game.GetScore()), 10, 10, 20, WHITE);
    
    // Draw health
    DrawText(TextFormat("Health: %d", game.GetPlayerHealth()), 10, 40, 20, RED);
    
    // Draw wave
    DrawText(TextFormat("Wave: %d", game.GetCurrentWave()), 10, 70, 20, YELLOW);
}
```

---

## How It Works

### The Magic Line

```cpp
static GameManager& Instance()
{
    static GameManager instance;  // ← This is the magic!
    return instance;
}
```

**What happens:**
1. First call: `instance` is created (constructor runs)
2. Subsequent calls: Same `instance` is returned
3. **static** means it lives for the entire program lifetime
4. **C++ guarantees** this is thread-safe (since C++11)

### Why Private Constructor?

```cpp
private:
    GameManager();  // ← Can't do: GameManager game;
```

This prevents:
```cpp
GameManager game1;  // ✗ Compile error!
GameManager game2;  // ✗ Compile error!
```

Forces you to use:
```cpp
GameManager& game = GameManager::Instance();  // ✓ Only way
```

### Why Delete Copy?

```cpp
GameManager(const GameManager&) = delete;
GameManager& operator=(const GameManager&) = delete;
```

Prevents:
```cpp
GameManager& original = GameManager::Instance();
GameManager copy = original;  // ✗ Compile error - can't copy!
```

---

## Exercise 1: Basic Extension - Debug Manager

**Goal:** Create a DebugManager singleton that tracks and displays debug information.

**Requirements:**
1. Create `DebugManager` class following singleton pattern
2. Track FPS (use Raylib's `GetFPS()`)
3. Track number of active bullets
4. Track number of active zombies
5. Add a `SetDebugMode(bool)` toggle
6. Add a `DrawDebugInfo()` method that shows this data

**Starter code:**

```cpp
class DebugManager
{
private:
    DebugManager();
    // TODO: Add copy prevention
    
    bool debugEnabled;
    int activeBullets;
    int activeZombies;
    
public:
    static DebugManager& Instance()
    {
        // TODO: Implement singleton pattern
    }
    
    // TODO: Add methods
    void SetDebugMode(bool enabled);
    void UpdateBulletCount(int count);
    void UpdateZombieCount(int count);
    void DrawDebugInfo();
};
```

**Expected output when debug enabled:**
```
FPS: 60
Bullets: 45
Zombies: 23
```

---

## Exercise 2: Intermediate Challenge - Settings System

**Goal:** Extend GameManager with a settings system.

**Add these settings:**
1. Master volume (0.0 to 1.0)
2. Difficulty multiplier (0.5 = easy, 1.0 = normal, 2.0 = hard)
3. Player movement speed
4. Show FPS toggle

**Requirements:**
1. Add getters and setters for each setting
2. Add a `SaveSettings()` method (can just print for now)
3. Add a `LoadSettings()` method (can use default values for now)
4. Difficulty should affect:
    - Zombie spawn rate
    - Zombie health
    - Score multiplier

**Hint:** Add these to GameManager:
```cpp
private:
    float masterVolume;
    float difficultyMultiplier;
    float playerSpeed;
    bool showFPS;
    
public:
    void SetDifficulty(float multiplier);
    float GetDifficulty() const;
    int ApplyDifficultyToHealth(int baseHealth) const;
    int ApplyDifficultyToScore(int baseScore) const;
```

**Test it:**
- Easy mode (0.5): Zombies have 50% health, score worth 200%
- Hard mode (2.0): Zombies have 200% health, score worth 50%

---

## Exercise 3: Advanced (Optional) - Multiple Singletons Communication

**Goal:** Make GameManager and DebugManager communicate.

**Scenario:** When player takes damage, DebugManager should flash the screen red for one frame.

**Requirements:**
1. Add `OnPlayerDamaged()` method to DebugManager
2. Make `GameManager::DamagePlayer()` notify DebugManager
3. DebugManager draws a red overlay that fades out over 0.5 seconds
4. Add a damage log showing last 5 damage events with timestamps

**Challenge question:** Is having two singletons talk to each other good design? What could go wrong?

---

## Common Mistakes

### ❌ Mistake 1: Creating multiple instances

```cpp
// WRONG - trying to create your own instance
GameManager* manager = new GameManager();  // Won't compile - constructor is private!
```

**Fix:** Always use `Instance()`
```cpp
// CORRECT
GameManager& manager = GameManager::Instance();
```

---

### ❌ Mistake 2: Storing pointers instead of references

```cpp
// WRONG - unnecessary pointer
GameManager* game = &GameManager::Instance();
game->AddScore(10);
```

**Fix:** Use reference
```cpp
// CORRECT
GameManager& game = GameManager::Instance();
game.AddScore(10);

// Or just use directly
GameManager::Instance().AddScore(10);
```

---

### ❌ Mistake 3: Forgetting to delete copy operations

```cpp
// WRONG - allows copying
class GameManager
{
public:
    static GameManager& Instance();
    // Oops! Forgot to delete copy constructor
};

// Now this is possible (BAD!):
GameManager copy = GameManager::Instance();  // Makes a copy!
```

**Fix:** Always delete copy operations
```cpp
GameManager(const GameManager&) = delete;
GameManager& operator=(const GameManager&) = delete;
```

---

### ❌ Mistake 4: Overusing Singleton

```cpp
// WRONG - making everything a singleton
class Player : public Singleton<Player> { };  // No! There's only one player, but use a regular class
class Bullet : public Singleton<Bullet> { };  // No! You need many bullets
class UI : public Singleton<UI> { };          // Maybe... but probably not necessary
```

**Fix:** Only use Singleton for truly global, single-instance systems
- ✓ GameManager (one game)
- ✓ AudioManager (one audio system)
- ✓ InputManager (one input handler)
- ✗ Player (use regular class)
- ✗ Bullet (use regular class)

---

## When to Use Singleton

### ✅ Good Use Cases:
- **Game-wide settings** (screen size, difficulty)
- **Score/stats tracking** (one score for the game)
- **Resource managers** (one texture manager, one audio manager)
- **Input handling** (one keyboard, one mouse)
- **Logging system** (one log file)

### ❌ Bad Use Cases:
- **When you need multiple instances** (players, enemies, bullets)
- **When local/stack variables work** (temporary data)
- **When you can pass parameters** (don't use singleton just to avoid passing data)
- **When testing** (singletons make unit testing harder)

### The Rule of Thumb:
**"If there should truly be only ONE in the entire game, consider Singleton."**

---

## Alternatives to Singleton

Sometimes people use Singleton when they shouldn't. Consider these alternatives:

### Alternative 1: Dependency Injection
```cpp
// Instead of accessing global singleton:
void Zombie::Update()
{
    GameManager::Instance().AddScore(10);  // Singleton
}

// Pass dependencies:
void Zombie::Update(GameManager& gameManager)
{
    gameManager.AddScore(10);  // Better for testing!
}
```

### Alternative 2: Static Functions Only
```cpp
// If you don't need state, just use static functions
class MathHelper
{
public:
    static float Distance(Vector2 a, Vector2 b);
    static float Lerp(float a, float b, float t);
    // No instance needed!
};
```

---

## Summary

✅ **Singleton** ensures one instance of a class<br>
✅ **Global access** from anywhere in the code<br>
✅ **Use for:** Game managers, settings, resource managers<br>
✅ **Don't overuse:** Not everything should be a singleton<br>
✅ **Implementation:** Private constructor + static Instance() method + delete copies

**The pattern you just learned:**
```cpp
class MySingleton
{
private:
    MySingleton() { }
    MySingleton(const MySingleton&) = delete;
    MySingleton& operator=(const MySingleton&) = delete;
    
public:
    static MySingleton& Instance()
    {
        static MySingleton instance;
        return instance;
    }
};
```

**Next Pattern:** Object Pool - because creating/destroying bullets 60 times per second is expensive!