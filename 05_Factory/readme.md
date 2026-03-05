# Factory Pattern

## The Problem

Your **Zombie Horde** game needs different zombie types. You start creating them directly:

```cpp
void SpawnZombies()
{
    // Spawn a slow zombie
    Zombie* slowZombie = new Zombie();
    slowZombie->SetSpeed(50.0f);
    slowZombie->SetHealth(50);
    slowZombie->SetColor(GREEN);
    slowZombie->SetDamage(10);
    slowZombie->SetSize(20.0f);
    slowZombie->SetPoints(10);
    
    // Spawn a fast zombie
    Zombie* fastZombie = new Zombie();
    fastZombie->SetSpeed(150.0f);
    fastZombie->SetHealth(20);
    fastZombie->SetColor(YELLOW);
    fastZombie->SetDamage(5);
    fastZombie->SetSize(15.0f);
    fastZombie->SetPoints(20);
    
    // Spawn a tank zombie
    Zombie* tankZombie = new Zombie();
    tankZombie->SetSpeed(30.0f);
    tankZombie->SetHealth(200);
    tankZombie->SetColor(RED);
    tankZombie->SetDamage(25);
    tankZombie->SetSize(30.0f);
    tankZombie->SetPoints(50);
}
```

### What's Wrong?

**Problem 1: Repetitive code**
```cpp
// Creating each zombie type requires 6-7 lines of setup
// Copy-paste everywhere you spawn zombies
// Easy to make mistakes (forgot to set speed, wrong color, etc.)
```

**Problem 2: Hard to maintain**
```cpp
// Want to balance the game? Change fast zombie health from 20 → 30?
// Now you need to find EVERY place you create fast zombies
// Miss one spot = inconsistent zombies!
```

**Problem 3: Type-based spawning is messy**
```cpp
void SpawnZombieByType(int type)
{
    if (type == 0)
    {
        // 7 lines for slow zombie
    }
    else if (type == 1)
    {
        // 7 lines for fast zombie
    }
    else if (type == 2)
    {
        // 7 lines for tank zombie
    }
    // Want to add exploding zombie? Add another else if...
}
```

**Problem 4: Scattered zombie definitions**
```cpp
// Slow zombie stats defined in SpawnSystem.cpp
// Fast zombie stats defined in WaveManager.cpp
// Tank zombie stats defined in BossSpawner.cpp
// Which one is the "real" definition?
```

---

## The Solution: Factory Pattern

**Centralize object creation in one place.**

Think of it like a **car factory**:
- You order a "sedan" or "truck" or "sports car"
- Factory knows exactly how to build each type
- You get a completed car, ready to drive
- All sedans are built the same way

### Key Characteristics:
1. **One place** to create all variants
2. **Encapsulates** creation logic
3. **Easy to extend** - add new types without changing existing code
4. **Consistent** - all zombies of the same type are identical

---

## Implementation in Zombie Horde

### ZombieType Enum

```cpp
#ifndef ZOMBIE_TYPE_H
#define ZOMBIE_TYPE_H

enum class ZombieType
{
    SLOW,      // Basic zombie - slow but common
    FAST,      // Runner - quick but weak
    TANK,      // Bullet sponge - slow but tough
    EXPLODER,  // Explodes on death
    BOSS       // Rare, powerful zombie
};

#endif
```

---

### Zombie.h (Base Class)

```cpp
#ifndef ZOMBIE_H
#define ZOMBIE_H

#include "raylib.h"

class Zombie
{
protected:
    Vector2 position;
    Vector2 velocity;
    float speed;
    int health;
    int maxHealth;
    int damage;
    float size;
    int pointValue;
    Color color;
    bool active;
    
public:
    Zombie();
    virtual ~Zombie() = default;
    
    // Lifecycle
    virtual void Activate(Vector2 pos);
    virtual void Deactivate();
    bool IsActive() const { return active; }
    
    // Update and Draw
    virtual void Update(float deltaTime, Vector2 playerPos);
    virtual void Draw() const;
    
    // Combat
    virtual void TakeDamage(int dmg);
    bool IsDead() const { return health <= 0; }
    int GetDamage() const { return damage; }
    int GetPointValue() const { return pointValue; }
    
    // Accessors
    Vector2 GetPosition() const { return position; }
    float GetSize() const { return size; }
    
    // Setup (called by factory)
    void SetStats(float spd, int hp, int dmg, float sz, int pts, Color col);
};

#endif
```

### Zombie.cpp

```cpp
#include "Zombie.h"
#include "GameManager.h"
#include <cmath>

Zombie::Zombie()
    : position({0, 0})
    , velocity({0, 0})
    , speed(50.0f)
    , health(50)
    , maxHealth(50)
    , damage(10)
    , size(20.0f)
    , pointValue(10)
    , color(GREEN)
    , active(false)
{
}

void Zombie::Activate(Vector2 pos)
{
    position = pos;
    health = maxHealth;
    active = true;
}

void Zombie::Deactivate()
{
    active = false;
}

void Zombie::SetStats(float spd, int hp, int dmg, float sz, int pts, Color col)
{
    speed = spd;
    health = hp;
    maxHealth = hp;
    damage = dmg;
    size = sz;
    pointValue = pts;
    color = col;
}

void Zombie::Update(float deltaTime, Vector2 playerPos)
{
    if (!active) return;
    
    // Move toward player
    Vector2 direction = {
        playerPos.x - position.x,
        playerPos.y - position.y
    };
    
    // Normalize direction
    float length = sqrtf(direction.x * direction.x + direction.y * direction.y);
    if (length > 0)
    {
        direction.x /= length;
        direction.y /= length;
    }
    
    // Move
    position.x += direction.x * speed * deltaTime;
    position.y += direction.y * speed * deltaTime;
}

void Zombie::Draw() const
{
    if (!active) return;
    
    // Draw zombie as circle
    DrawCircleV(position, size, color);
    
    // Draw health bar
    float barWidth = size * 2;
    float barHeight = 5;
    float healthPercent = (float)health / (float)maxHealth;
    
    DrawRectangle(position.x - barWidth/2, position.y - size - 10, 
                  barWidth, barHeight, RED);
    DrawRectangle(position.x - barWidth/2, position.y - size - 10, 
                  barWidth * healthPercent, barHeight, GREEN);
}

void Zombie::TakeDamage(int dmg)
{
    health -= dmg;
    if (health <= 0)
    {
        health = 0;
        active = false;
    }
}
```

---

### ZombieFactory.h

```cpp
#ifndef ZOMBIE_FACTORY_H
#define ZOMBIE_FACTORY_H

#include "Zombie.h"
#include "ZombieType.h"

class ZombieFactory
{
public:
    // Create a zombie of the specified type
    static Zombie* CreateZombie(ZombieType type);
    
    // Create a zombie at a specific position
    static Zombie* CreateZombieAt(ZombieType type, Vector2 position);
    
    // Get random zombie type (for variety)
    static ZombieType GetRandomType();
    
    // Get random type based on wave (difficulty scaling)
    static ZombieType GetRandomTypeForWave(int wave);
    
private:
    // Internal: Configure zombie stats based on type
    static void ConfigureZombie(Zombie* zombie, ZombieType type);
};

#endif
```

### ZombieFactory.cpp

```cpp
#include "ZombieFactory.h"
#include <cstdlib>

Zombie* ZombieFactory::CreateZombie(ZombieType type)
{
    Zombie* zombie = new Zombie();
    ConfigureZombie(zombie, type);
    return zombie;
}

Zombie* ZombieFactory::CreateZombieAt(ZombieType type, Vector2 position)
{
    Zombie* zombie = CreateZombie(type);
    zombie->Activate(position);
    return zombie;
}

void ZombieFactory::ConfigureZombie(Zombie* zombie, ZombieType type)
{
    // All zombie configurations in ONE place!
    switch (type)
    {
        case ZombieType::SLOW:
            zombie->SetStats(
                50.0f,      // speed
                50,         // health
                10,         // damage
                20.0f,      // size
                10,         // points
                GREEN       // color
            );
            break;
            
        case ZombieType::FAST:
            zombie->SetStats(
                150.0f,     // speed - 3x faster!
                20,         // health - fragile
                5,          // damage - weak
                15.0f,      // size - smaller
                20,         // points - worth more
                YELLOW      // color
            );
            break;
            
        case ZombieType::TANK:
            zombie->SetStats(
                30.0f,      // speed - very slow
                200,        // health - bullet sponge!
                25,         // damage - hits hard
                30.0f,      // size - big and scary
                50,         // points - big reward
                RED         // color
            );
            break;
            
        case ZombieType::EXPLODER:
            zombie->SetStats(
                80.0f,      // speed - medium fast
                30,         // health - fragile
                50,         // damage - huge on death!
                18.0f,      // size
                30,         // points
                ORANGE      // color
            );
            break;
            
        case ZombieType::BOSS:
            zombie->SetStats(
                40.0f,      // speed - slow but unstoppable
                500,        // health - mini-boss
                40,         // damage - devastating
                50.0f,      // size - HUGE
                200,        // points - jackpot!
                PURPLE      // color
            );
            break;
    }
}

ZombieType ZombieFactory::GetRandomType()
{
    int random = GetRandomValue(0, 100);
    
    // Weighted random:
    // 60% Slow
    // 25% Fast
    // 15% Tank
    if (random < 60)
        return ZombieType::SLOW;
    else if (random < 85)
        return ZombieType::FAST;
    else
        return ZombieType::TANK;
}

ZombieType ZombieFactory::GetRandomTypeForWave(int wave)
{
    int random = GetRandomValue(0, 100);
    
    // Wave 1-3: Only slow zombies
    if (wave <= 3)
    {
        return ZombieType::SLOW;
    }
    
    // Wave 4-6: Slow + Fast
    if (wave <= 6)
    {
        return (random < 70) ? ZombieType::SLOW : ZombieType::FAST;
    }
    
    // Wave 7-9: Slow + Fast + Tank
    if (wave <= 9)
    {
        if (random < 50)
            return ZombieType::SLOW;
        else if (random < 80)
            return ZombieType::FAST;
        else
            return ZombieType::TANK;
    }
    
    // Wave 10+: All types + Boss chance
    if (random < 35)
        return ZombieType::SLOW;
    else if (random < 65)
        return ZombieType::FAST;
    else if (random < 90)
        return ZombieType::TANK;
    else if (random < 97)
        return ZombieType::EXPLODER;
    else
        return ZombieType::BOSS;  // 3% boss chance!
}
```

---

## How to Use It

### Simple Usage - Create Specific Type

```cpp
#include "ZombieFactory.h"

void SpawnWave()
{
    // Create 5 slow zombies
    for (int i = 0; i < 5; i++)
    {
        Vector2 spawnPos = GetRandomSpawnPosition();
        Zombie* zombie = ZombieFactory::CreateZombieAt(ZombieType::SLOW, spawnPos);
        zombies.push_back(zombie);
    }
    
    // Create 2 fast zombies
    for (int i = 0; i < 2; i++)
    {
        Vector2 spawnPos = GetRandomSpawnPosition();
        Zombie* zombie = ZombieFactory::CreateZombieAt(ZombieType::FAST, spawnPos);
        zombies.push_back(zombie);
    }
    
    // Create 1 tank zombie
    Zombie* tank = ZombieFactory::CreateZombieAt(ZombieType::TANK, GetRandomSpawnPosition());
    zombies.push_back(tank);
}
```

### Random Spawning

```cpp
void SpawnRandomZombie()
{
    ZombieType type = ZombieFactory::GetRandomType();
    Vector2 pos = GetRandomSpawnPosition();
    
    Zombie* zombie = ZombieFactory::CreateZombieAt(type, pos);
    zombies.push_back(zombie);
}
```

### Wave-Based Difficulty

```cpp
void SpawnWaveZombies(int wave)
{
    int zombieCount = 5 + (wave * 2);  // More zombies each wave
    
    for (int i = 0; i < zombieCount; i++)
    {
        // Get appropriate type for this wave
        ZombieType type = ZombieFactory::GetRandomTypeForWave(wave);
        
        Vector2 pos = GetRandomSpawnPosition();
        Zombie* zombie = ZombieFactory::CreateZombieAt(type, pos);
        zombies.push_back(zombie);
    }
}
```

### With Object Pool (Best Practice)

```cpp
class ZombiePool
{
private:
    std::vector<Zombie> pool;
    
public:
    ZombiePool(int size) { pool.resize(size); }
    
    Zombie* Spawn(ZombieType type, Vector2 position)
    {
        // Find inactive zombie
        for (Zombie& zombie : pool)
        {
            if (!zombie.IsActive())
            {
                // Use factory to configure it
                ZombieFactory::ConfigureZombie(&zombie, type);
                zombie.Activate(position);
                return &zombie;
            }
        }
        return nullptr;
    }
};
```

---

## How It Works

### Before Factory (Scattered Creation)

```
SpawnSystem.cpp:
    Zombie z;
    z.SetSpeed(50);
    z.SetHealth(50);
    ...

WaveManager.cpp:
    Zombie z;
    z.SetSpeed(50);  // Same stats, duplicated!
    z.SetHealth(50);
    ...

BossSpawner.cpp:
    Zombie z;
    z.SetSpeed(50);  // Again!
    z.SetHealth(50);
    ...
```

### After Factory (Centralized Creation)

```
ZombieFactory.cpp:
    ┌─────────────────────────┐
    │ ALL zombie stats here!  │
    │ - SLOW: 50 speed, 50 hp │
    │ - FAST: 150 speed, 20hp │
    │ - TANK: 30 speed, 200hp │
    └─────────────────────────┘
              ↑
              │ Everyone uses the factory
              │
    ┌─────────┴──────────┬──────────┐
    │                    │          │
SpawnSystem        WaveManager   BossSpawner
CreateZombie()     CreateZombie() CreateZombie()
```

---

## Comparison: Before vs After

### Before Factory
```cpp
// To create a slow zombie (7 lines):
Zombie* z = new Zombie();
z->SetSpeed(50.0f);
z->SetHealth(50);
z->SetColor(GREEN);
z->SetDamage(10);
z->SetSize(20.0f);
z->SetPoints(10);

// To balance slow zombies (change health 50 → 60):
// - Find ALL places slow zombies are created
// - Change health in each place
// - Pray you didn't miss any
```

### After Factory
```cpp
// To create a slow zombie (1 line):
Zombie* z = ZombieFactory::CreateZombie(ZombieType::SLOW);

// To balance slow zombies (change health 50 → 60):
// - Open ZombieFactory.cpp
// - Change one number in ConfigureZombie()
// - Done! All slow zombies updated everywhere
```

---

## Exercise 1: Basic Extension - New Zombie Type

**Goal:** Add a HEALER zombie that heals nearby zombies.

**Requirements:**
1. Add `HEALER` to `ZombieType` enum
2. Stats for healer zombie:
    - Speed: 40
    - Health: 80
    - Damage: 0 (doesn't attack)
    - Size: 22
    - Points: 40
    - Color: SKYBLUE
3. Add case to `ConfigureZombie()`
4. Add healer to weighted random (5% chance)
5. Test: Spawn healer and verify it has correct stats

**Bonus:** Make healer actually heal nearby zombies in `Zombie::Update()`

---

## Exercise 2: Intermediate Challenge - Difficulty Scaling

**Goal:** Make zombie stats scale with difficulty setting.

**Requirements:**
1. Add difficulty multiplier to GameManager (0.5 = easy, 1.0 = normal, 2.0 = hard)
2. Modify `ConfigureZombie()` to apply difficulty:
    - Health *= difficulty
    - Damage *= difficulty
    - Points *= difficulty
    - Speed unchanged (would be frustrating)
3. Add `CreateZombieWithDifficulty(ZombieType type, float difficulty)` method
4. Test: Same zombie type should be tougher on hard mode

**Example:**
```cpp
// Normal: Tank has 200 health
// Hard (2.0x): Tank has 400 health
```

---

## Exercise 3: Advanced - Factory Method Pattern

**Goal:** Create subclass factories for zombie variants.

**Current:** All zombies share one Zombie class
**New:** SlowZombie, FastZombie, TankZombie each have unique behaviors

**Requirements:**
1. Create `SlowZombie : public Zombie` class
    - Override `Update()` - sometimes stops moving (idle behavior)
2. Create `FastZombie : public Zombie` class
    - Override `Update()` - zigzag movement pattern
3. Create `TankZombie : public Zombie` class
    - Override `TakeDamage()` - reduce damage by 50%
4. Modify factory to return correct subclass:
```cpp
Zombie* ZombieFactory::CreateZombie(ZombieType type)
{
    switch (type)
    {
        case ZombieType::SLOW:
            return new SlowZombie();
        case ZombieType::FAST:
            return new FastZombie();
        case ZombieType::TANK:
            return new TankZombie();
    }
}
```

**Challenge:** How does this work with Object Pool? (Hint: you can't pool different types together easily)

---

## Exercise 4: Advanced (Optional) - Data-Driven Factory

**Goal:** Load zombie stats from a file instead of hard-coding.

**Create:** `zombie_types.txt`
```
SLOW 50.0 50 10 20.0 10 0 255 0 255
FAST 150.0 20 5 15.0 20 255 255 0 255
TANK 30.0 200 25 30.0 50 255 0 0 255
```
Format: `TYPE SPEED HEALTH DAMAGE SIZE POINTS R G B A`

**Requirements:**
1. Create `LoadZombieStats(const char* filename)` function
2. Parse the file and store stats in a map
3. `ConfigureZombie()` looks up stats from map instead of switch
4. Now designers can balance the game by editing the text file!

**Bonus:** Support comments (#) and reload at runtime

---

## Common Mistakes

### ❌ Mistake 1: Not using the factory consistently

```cpp
// WRONG - mixing factory and manual creation
Zombie* z1 = ZombieFactory::CreateZombie(ZombieType::SLOW);  // Factory
Zombie* z2 = new Zombie();  // Manual!
z2->SetSpeed(50);  // Duplicating factory logic!
```

**Fix:** Always use the factory
```cpp
// CORRECT - all zombies come from factory
Zombie* z1 = ZombieFactory::CreateZombie(ZombieType::SLOW);
Zombie* z2 = ZombieFactory::CreateZombie(ZombieType::SLOW);
```

---

### ❌ Mistake 2: Making factory methods non-static unnecessarily

```cpp
// WRONG - why would you need an instance?
class ZombieFactory
{
public:
    Zombie* CreateZombie(ZombieType type);  // Non-static
};

// Now you need:
ZombieFactory factory;
Zombie* z = factory.CreateZombie(ZombieType::SLOW);  // Extra step!
```

**Fix:** Make it static (unless you need state)
```cpp
// CORRECT - static methods
class ZombieFactory
{
public:
    static Zombie* CreateZombie(ZombieType type);
};

// Clean usage:
Zombie* z = ZombieFactory::CreateZombie(ZombieType::SLOW);
```

---

### ❌ Mistake 3: Forgetting to configure the zombie

```cpp
Zombie* ZombieFactory::CreateZombie(ZombieType type)
{
    Zombie* zombie = new Zombie();
    // Oops! Forgot to call ConfigureZombie()!
    return zombie;  // Returns zombie with default stats!
}
```

**Fix:** Always configure before returning
```cpp
Zombie* ZombieFactory::CreateZombie(ZombieType type)
{
    Zombie* zombie = new Zombie();
    ConfigureZombie(zombie, type);  // Essential!
    return zombie;
}
```

---

### ❌ Mistake 4: Giant switch with duplicated code

```cpp
// WRONG - lots of duplication
void ConfigureZombie(Zombie* z, ZombieType type)
{
    switch (type)
    {
        case SLOW:
            z->SetSpeed(50.0f);
            z->SetHealth(50);
            z->SetDamage(10);
            z->SetSize(20.0f);
            z->SetPoints(10);
            z->SetColor(GREEN);
            break;
        case FAST:
            z->SetSpeed(150.0f);    // 6 lines repeated
            z->SetHealth(20);
            z->SetDamage(5);
            z->SetSize(15.0f);
            z->SetPoints(20);
            z->SetColor(YELLOW);
            break;
        // ... more duplication
    }
}
```

**Fix:** Use a single SetStats method
```cpp
// CORRECT - one call per type
void ConfigureZombie(Zombie* z, ZombieType type)
{
    switch (type)
    {
        case SLOW:
            z->SetStats(50.0f, 50, 10, 20.0f, 10, GREEN);
            break;
        case FAST:
            z->SetStats(150.0f, 20, 5, 15.0f, 20, YELLOW);
            break;
        // Much cleaner!
    }
}
```

---

### ❌ Mistake 5: Memory management confusion with factory

```cpp
// Creates zombie with new
Zombie* z = ZombieFactory::CreateZombie(ZombieType::SLOW);

// ... later ...

// WRONG - who deletes it? Factory? User?
// Memory leak if nobody deletes!
```

**Fix:** Be explicit about ownership
```cpp
// Option 1: Factory returns raw pointer, user owns it
Zombie* z = ZombieFactory::CreateZombie(ZombieType::SLOW);
// User must delete later
delete z;

// Option 2: Factory works with pools (no new/delete)
class ZombieFactory
{
public:
    static void ConfigureZombie(Zombie* zombie, ZombieType type);
    // Zombie already exists in pool, we just configure it
};

// Pool owns the memory
zombiePool.Spawn(ZombieType::SLOW, position);
```

---

## When to Use Factory Pattern

### ✅ Good Use Cases:
- **Multiple variants** of the same object (zombie types, weapon types, enemy types)
- **Complex creation** with many parameters
- **Centralized configuration** (balancing, tuning)
- **Type-based spawning** (spawn by ID, name, or enum)
- **Difficulty scaling** (modify stats based on settings)

### ❌ Bad Use Cases:
- **Only one type** (just create it directly)
- **Simple objects** with 1-2 parameters
- **Unique instances** (player, boss - just make them directly)
- **Already using pools** (pools handle creation, factory just configures)

### The Rule of Thumb:
**"If you have 3+ variants with different configurations, use a Factory."**

---

## Factory Pattern Variations

### Simple Factory (What we built)
```cpp
static Zombie* CreateZombie(ZombieType type);
```
- One factory class
- Switch/if statement
- Good for small projects

### Factory Method Pattern
```cpp
class ZombieFactory
{
public:
    virtual Zombie* CreateZombie() = 0;
};

class SlowZombieFactory : public ZombieFactory { /* ... */ };
class FastZombieFactory : public ZombieFactory { /* ... */ };
```
- Subclasses for each type
- More flexible but more complex
- Good for large projects

### Abstract Factory Pattern
```cpp
class EnemyFactory
{
public:
    virtual Zombie* CreateZombie() = 0;
    virtual Demon* CreateDemon() = 0;
    virtual Ghost* CreateGhost() = 0;
};
```
- Creates families of related objects
- Very flexible, very complex
- Good for large games with themed levels

---

## Combining Factory with Other Patterns

### Factory + Object Pool
```cpp
// Pool creates zombies once
ZombiePool pool(100);

// Factory configures them when spawned
Zombie* zombie = pool.GetInactive();
if (zombie)
{
    ZombieFactory::ConfigureZombie(zombie, ZombieType::FAST);
    zombie->Activate(position);
}
```

### Factory + Singleton
```cpp
class ZombieFactory
{
private:
    ZombieFactory() { LoadStats(); }
    // ... singleton pattern ...
    
public:
    static ZombieFactory& Instance();
    Zombie* CreateZombie(ZombieType type);
};

// Usage:
Zombie* z = ZombieFactory::Instance().CreateZombie(ZombieType::SLOW);
```

### Factory + Strategy Pattern
```cpp
// Factory creates zombie with strategy
Zombie* zombie = ZombieFactory::CreateZombie(ZombieType::FAST);
zombie->SetMovementStrategy(new ZigZagStrategy());  // Fast zombies zigzag!
```

---

## Summary

✅ **Factory Pattern** centralizes object creation<br>
✅ **One place** for all configurations<br>
✅ **Easy to balance** - change stats in one place<br>
✅ **Easy to extend** - add new types easily<br>
✅ **Type-safe** - use enums instead of strings/numbers

**The pattern you just learned:**
```cpp
enum class Type { TYPE_A, TYPE_B, TYPE_C };

class Factory
{
public:
    static Object* Create(Type type)
    {
        Object* obj = new Object();
        Configure(obj, type);
        return obj;
    }
    
private:
    static void Configure(Object* obj, Type type)
    {
        switch (type)
        {
            case TYPE_A: obj->SetStats(...); break;
            case TYPE_B: obj->SetStats(...); break;
            case TYPE_C: obj->SetStats(...); break;
        }
    }
};
```

**Before Factory:**
```cpp
// To create slow zombie:
Zombie* z = new Zombie();
z->SetSpeed(50);
z->SetHealth(50);
z->SetColor(GREEN);
z->SetDamage(10);
z->SetSize(20);
z->SetPoints(10);

// Repeated 20x across the codebase!
```

**After Factory:**
```cpp
// To create slow zombie:
Zombie* z = ZombieFactory::CreateZombie(ZombieType::SLOW);

// To balance (change slow zombie health):
// Open ZombieFactory.cpp → Change ONE number → Done!
```

**Next Pattern:** Observer Pattern - handling events cleanly (OnZombieKilled, OnPlayerHit, etc.)!