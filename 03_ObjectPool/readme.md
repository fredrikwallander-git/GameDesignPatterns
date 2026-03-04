# Object Pool Pattern

## The Problem

You've got the player shooting bullets in **Zombie Horde**. Every time the player presses SPACE, you create a new bullet:

```cpp
void Player::Shoot()
{
    Bullet* bullet = new Bullet(position, direction);
    bullets.push_back(bullet);
}
```

And when bullets go off-screen, you delete them:

```cpp
void UpdateBullets()
{
    for (int i = 0; i < bullets.size(); i++)
    {
        if (bullets[i]->IsOffScreen())
        {
            delete bullets[i];  // Free memory
            bullets.erase(bullets.begin() + i);
            i--;
        }
    }
}
```

**The game runs at 60 FPS. Player shoots 10 bullets per second.**

That's **600 new/delete operations per second!**

### What Goes Wrong:

**Problem 1: Performance**
- `new` and `delete` are **slow** (memory allocation takes time)
- Garbage collector has to clean up (causes frame drops)
- Memory gets **fragmented** (Swiss cheese effect)

**Problem 2: Frame Drops**
```
Frame 1: Create 5 bullets, delete 3 bullets → 60 FPS
Frame 2: Create 10 bullets, delete 8 bullets → 55 FPS ← Lag spike!
Frame 3: Create 15 bullets, delete 12 bullets → 45 FPS ← Worse!
```

Players notice the **stutter**.

**Problem 3: Memory Fragmentation**
```
Memory: [Bullet][Free][Bullet][Free][Free][Bullet][Free]
                ↑ Holes everywhere - inefficient!
```

---

## The Solution: Object Pool Pattern

**Instead of creating/destroying objects, REUSE them!**

Think of it like a **library**:
- Library has 100 books (objects)
- You **check out** a book when you need it (activate)
- You **return** the book when done (deactivate)
- Books are never destroyed, just reused

### Key Characteristics:
1. **Pre-allocate** all objects at startup (one-time cost)
2. **Activate** objects from the pool when needed
3. **Deactivate** objects back to the pool when done
4. **Never** create or destroy during gameplay

---

## Implementation in Zombie Horde

### Bullet.h

```cpp
#ifndef BULLET_H
#define BULLET_H

#include "raylib.h"

class Bullet
{
private:
    Vector2 position;
    Vector2 velocity;
    float speed;
    float radius;
    bool active;  // ← KEY: Is this bullet currently in use?
    
public:
    Bullet();
    
    // Activation/Deactivation
    void Activate(Vector2 startPos, Vector2 direction, float bulletSpeed);
    void Deactivate();
    bool IsActive() const { return active; }
    
    // Update and Draw
    void Update(float deltaTime);
    void Draw() const;
    
    // Collision
    Vector2 GetPosition() const { return position; }
    float GetRadius() const { return radius; }
    bool IsOffScreen() const;
};

#endif
```

### Bullet.cpp

```cpp
#include "Bullet.h"
#include "GameManager.h"

Bullet::Bullet()
    : position({0, 0})
    , velocity({0, 0})
    , speed(500.0f)
    , radius(5.0f)
    , active(false)  // Starts inactive
{
}

void Bullet::Activate(Vector2 startPos, Vector2 direction, float bulletSpeed)
{
    position = startPos;
    velocity = direction;
    speed = bulletSpeed;
    active = true;
}

void Bullet::Deactivate()
{
    active = false;
}

void Bullet::Update(float deltaTime)
{
    if (!active) return;  // Skip if not active
    
    // Move bullet
    position.x += velocity.x * speed * deltaTime;
    position.y += velocity.y * speed * deltaTime;
    
    // Deactivate if off-screen
    if (IsOffScreen())
    {
        Deactivate();
    }
}

void Bullet::Draw() const
{
    if (!active) return;  // Don't draw if not active
    
    DrawCircleV(position, radius, YELLOW);
}

bool Bullet::IsOffScreen() const
{
    return position.x < -radius || 
           position.x > GameManager::SCREEN_WIDTH + radius ||
           position.y < -radius || 
           position.y > GameManager::SCREEN_HEIGHT + radius;
}
```

---

### BulletPool.h

```cpp
#ifndef BULLET_POOL_H
#define BULLET_POOL_H

#include "Bullet.h"
#include <vector>

class BulletPool
{
private:
    std::vector<Bullet> pool;  // Pre-allocated bullets
    int poolSize;
    
public:
    BulletPool(int size = 100);  // Create pool with 100 bullets
    
    // Get an inactive bullet and activate it
    Bullet* Spawn(Vector2 position, Vector2 direction, float speed);
    
    // Update and draw all active bullets
    void UpdateAll(float deltaTime);
    void DrawAll() const;
    
    // Stats
    int GetActiveCount() const;
    int GetPoolSize() const { return poolSize; }
    
    // Collision checking
    Bullet* GetActiveBullet(int index);
    int GetActiveBulletCount() const { return GetActiveCount(); }
};

#endif
```

### BulletPool.cpp

```cpp
#include "BulletPool.h"

BulletPool::BulletPool(int size)
    : poolSize(size)
{
    // Pre-allocate ALL bullets at once
    pool.resize(poolSize);
    
    // All bullets start inactive (ready to use)
}

Bullet* BulletPool::Spawn(Vector2 position, Vector2 direction, float speed)
{
    // Find first inactive bullet
    for (int i = 0; i < poolSize; i++)
    {
        if (!pool[i].IsActive())
        {
            // Found one! Activate it and return
            pool[i].Activate(position, direction, speed);
            return &pool[i];
        }
    }
    
    // Pool exhausted - no bullets available
    // Option 1: Return nullptr (what we'll do)
    // Option 2: Deactivate oldest bullet and reuse it
    // Option 3: Expand the pool (defeats the purpose)
    return nullptr;
}

void BulletPool::UpdateAll(float deltaTime)
{
    // Update all active bullets
    for (int i = 0; i < poolSize; i++)
    {
        if (pool[i].IsActive())
        {
            pool[i].Update(deltaTime);
        }
    }
}

void BulletPool::DrawAll() const
{
    // Draw all active bullets
    for (int i = 0; i < poolSize; i++)
    {
        if (pool[i].IsActive())
        {
            pool[i].Draw();
        }
    }
}

int BulletPool::GetActiveCount() const
{
    int count = 0;
    for (int i = 0; i < poolSize; i++)
    {
        if (pool[i].IsActive())
        {
            count++;
        }
    }
    return count;
}

Bullet* BulletPool::GetActiveBullet(int index)
{
    int activeIndex = 0;
    for (int i = 0; i < poolSize; i++)
    {
        if (pool[i].IsActive())
        {
            if (activeIndex == index)
            {
                return &pool[i];
            }
            activeIndex++;
        }
    }
    return nullptr;
}
```

---

## How to Use It

### In main.cpp

```cpp
#include "BulletPool.h"

int main()
{
    InitWindow(800, 600, "Zombie Horde");
    SetTargetFPS(60);
    
    // Create bullet pool ONCE at startup
    BulletPool bulletPool(100);  // Pool of 100 bullets
    
    Player player;
    
    while (!WindowShouldClose())
    {
        float deltaTime = GetFrameTime();
        
        // Player shoots
        if (IsKeyPressed(KEY_SPACE))
        {
            Vector2 direction = {1, 0};  // Shoot right
            
            // Get bullet from pool instead of creating new
            Bullet* bullet = bulletPool.Spawn(player.GetPosition(), direction, 500.0f);
            
            if (bullet == nullptr)
            {
                // Pool exhausted - maybe play "out of ammo" sound
            }
        }
        
        // Update all bullets
        bulletPool.UpdateAll(deltaTime);
        
        // Draw
        BeginDrawing();
        ClearBackground(DARKGRAY);
        
        player.Draw();
        bulletPool.DrawAll();
        
        // Debug info
        DrawText(TextFormat("Active Bullets: %d / %d", 
            bulletPool.GetActiveCount(), 
            bulletPool.GetPoolSize()), 
            10, 10, 20, WHITE);
        
        EndDrawing();
    }
    
    CloseWindow();
    return 0;
}
```

---

## How It Works

### Memory Layout - Before (Dynamic Allocation)

```
Frame 1:
Heap: [Bullet1] [Free] [Bullet2] [Free]
      ↑ new          ↑ new

Frame 2:
Heap: [Free] [Free] [Bullet3] [Free] [Bullet4]
      ↑ delete       ↑ new            ↑ new

Frame 3:
Heap: [Bullet5] [Free] [Free] [Free] [Bullet6]
      ↑ new                             ↑ new
      
Memory is fragmented, allocation is slow!
```

### Memory Layout - After (Object Pool)

```
Initialization:
Pool: [Bullet0][Bullet1][Bullet2]...[Bullet99]
       inactive inactive inactive     inactive
       ↑ All allocated ONCE

Frame 1:
Pool: [Bullet0][Bullet1][Bullet2]...[Bullet99]
       ACTIVE  ACTIVE   inactive     inactive
       ↑ Just flip flag to true!

Frame 2:
Pool: [Bullet0][Bullet1][Bullet2]...[Bullet99]
       inactive ACTIVE  ACTIVE       inactive
       ↑ Flip to false when done
       
Memory is contiguous, access is FAST!
```

### The Activation Cycle

```cpp
// 1. START: Bullet is inactive (in pool, waiting)
active = false;

// 2. SPAWN: Player shoots, bullet is activated
bulletPool.Spawn(position, direction, speed);
  → Find inactive bullet
  → Set position, velocity
  → active = true

// 3. ALIVE: Bullet flies across screen
Update() → move bullet
Draw() → render bullet

// 4. DEACTIVATE: Bullet goes off-screen or hits enemy
active = false;  ← Back to pool!

// 5. Ready to be reused!
```

---

## Performance Comparison

### Without Pool (Dynamic Allocation)
```
Bullets fired per second: 10
Operations: 10 new + 10 delete = 20 allocations/sec
At 60 FPS: 1,200 allocations per minute
Frame time: ~18ms (can spike to 25ms)
```

### With Pool
```
Bullets fired per second: 10
Operations: 0 allocations (just flip bool)
At 60 FPS: 0 allocations per minute
Frame time: ~16ms (stable)
```

**Result: 12% performance improvement + no frame drops!**

---

## Exercise 1: Basic Extension - Pool Statistics

**Goal:** Add detailed statistics to the BulletPool.

**Requirements:**
1. Track total bullets spawned (lifetime)
2. Track total bullets that went off-screen
3. Track peak active bullets (max at any one time)
4. Add a `PrintStats()` method
5. Add a `ResetStats()` method

**Add to BulletPool.h:**
```cpp
private:
    int totalSpawned;
    int totalDeactivated;
    int peakActive;
    
public:
    void PrintStats() const;
    void ResetStats();
```

**Expected output:**
```
=== Bullet Pool Stats ===
Pool Size: 100
Active: 23
Total Spawned: 1,547
Total Deactivated: 1,524
Peak Active: 67
Pool Usage: 67%
```

---

## Exercise 2: Intermediate Challenge - Particle Pool

**Goal:** Create a ParticlePool for blood splatter effects when zombies die.

**Requirements:**
1. Create `Particle` class
    - Position, velocity, color, lifetime, age
    - `Activate(position, velocity, color, lifetime)`
    - `Update()` - move and age
    - `Draw()` - draw with alpha based on age
2. Create `ParticlePool` (size = 500)
3. When zombie dies, spawn 5-10 particles in random directions
4. Particles fade out over 1 second

**Starter code:**

```cpp
class Particle
{
private:
    Vector2 position;
    Vector2 velocity;
    Color color;
    float lifetime;    // How long particle lives (seconds)
    float age;         // How old is it (seconds)
    bool active;
    
public:
    void Activate(Vector2 pos, Vector2 vel, Color col, float life);
    void Update(float deltaTime);
    void Draw() const;
    bool IsActive() const { return active; }
};
```

**Test it:**
- Kill zombie → blood splatter particles fly out
- Particles should fade from RED (alpha 255) to transparent (alpha 0)
- After 1 second, particles deactivate automatically

---

## Exercise 3: Advanced (Optional) - Smart Pool Recycling

**Goal:** When pool is exhausted, automatically recycle the oldest bullet.

**Current behavior:**
```cpp
Bullet* bullet = bulletPool.Spawn(...);
if (bullet == nullptr)
{
    // Pool full - can't shoot!
}
```

**New behavior:**
- Track the age of each active bullet
- When pool is full, deactivate the oldest bullet and reuse it
- Player can always shoot, but oldest bullets disappear

**Requirements:**
1. Add `float activeTime` to Bullet
2. Track how long each bullet has been active
3. In `Spawn()`, if pool full, find oldest active bullet
4. Deactivate oldest, activate at new position
5. Add `SetRecycleMode(bool)` to toggle this feature

**Challenge question:** What's the downside of this approach? When would it feel bad to the player?

---

## Exercise 4: Advanced (Optional) - Multiple Pools

**Goal:** Create pools for different bullet types.

**Scenario:**
- Player bullets (yellow, fast)
- Enemy bullets (red, slow)
- Power-up bullets (green, large)

**Two approaches:**

**Approach 1: Separate pools**
```cpp
BulletPool playerBullets(100);
BulletPool enemyBullets(200);
BulletPool powerBullets(50);
```

**Approach 2: Generic pool**
```cpp
template<typename T>
class ObjectPool
{
private:
    std::vector<T> pool;
    
public:
    ObjectPool(int size);
    T* Spawn();
    void UpdateAll(float deltaTime);
    void DrawAll() const;
};

// Usage:
ObjectPool<Bullet> bullets(100);
ObjectPool<Particle> particles(500);
ObjectPool<Zombie> zombies(50);  // Can even pool zombies!
```

Implement either approach and compare pros/cons.

---

## Common Mistakes

### ❌ Mistake 1: Forgetting to check if spawn succeeded

```cpp
// WRONG - bullet might be nullptr!
Bullet* bullet = bulletPool.Spawn(pos, dir, speed);
bullet->SetDamage(10);  // CRASH if pool was full!
```

**Fix:** Always check for null
```cpp
// CORRECT
Bullet* bullet = bulletPool.Spawn(pos, dir, speed);
if (bullet != nullptr)
{
    bullet->SetDamage(10);
}
```

---

### ❌ Mistake 2: Pool too small

```cpp
BulletPool bulletPool(10);  // Only 10 bullets?!

// Player fires 5 bullets per second
// After 2 seconds, pool is full
// Player can't shoot anymore!
```

**Fix:** Size pool based on max expected usage
```cpp
// Player fires 10/sec, bullets live ~2 seconds
// Max bullets on screen: 10 * 2 = 20
// Add safety margin: 20 * 1.5 = 30
BulletPool bulletPool(50);  // Comfortable margin
```

**Formula:** `poolSize = fireRate * bulletLifetime * safetyMargin`

---

### ❌ Mistake 3: Not resetting all data on activate

```cpp
void Bullet::Activate(Vector2 pos, Vector2 dir, float spd)
{
    position = pos;
    active = true;
    // Oops! Forgot to reset velocity and speed!
    // Bullet will use old values from last time!
}
```

**Fix:** Reset ALL state
```cpp
void Bullet::Activate(Vector2 pos, Vector2 dir, float spd)
{
    position = pos;
    velocity = dir;
    speed = spd;
    active = true;
    damage = 10;  // Reset everything!
    color = YELLOW;
    // etc.
}
```

---

### ❌ Mistake 4: Deleting pooled objects

```cpp
// WRONG - these are in the pool, don't delete them!
for (Bullet* bullet : activeBullets)
{
    delete bullet;  // NO! These aren't dynamically allocated!
}
```

**Fix:** Just deactivate
```cpp
// CORRECT
for (int i = 0; i < pool.size(); i++)
{
    pool[i].Deactivate();  // Just flip the flag
}
```

---

### ❌ Mistake 5: Pool as pointer instead of value

```cpp
// WRONG - unnecessary dynamic allocation
std::vector<Bullet*> pool;

BulletPool::BulletPool(int size)
{
    for (int i = 0; i < size; i++)
    {
        pool.push_back(new Bullet());  // Still doing new/delete!
    }
}
```

**Fix:** Store by value
```cpp
// CORRECT - all bullets in contiguous memory
std::vector<Bullet> pool;

BulletPool::BulletPool(int size)
{
    pool.resize(size);  // One allocation, all bullets ready
}
```

---

## When to Use Object Pool

### ✅ Good Use Cases:
- **Bullets/projectiles** - created and destroyed frequently
- **Particles** - hundreds spawned per second
- **Enemies** - spawn in waves, die, respawn
- **Audio sources** - play sound, return to pool
- **UI elements** - damage numbers, notifications
- **Temporary effects** - explosions, muzzle flashes

### ❌ Bad Use Cases:
- **Player** - only one, never destroyed
- **Level geometry** - created once, never destroyed
- **Persistent NPCs** - live for entire game
- **Singletons** - by definition, only one instance
- **Very large objects** - pool wastes too much memory

### The Rule of Thumb:
**"If you create/destroy the same type of object more than 10 times per second, pool it."**

---

## Performance Tips

### Tip 1: Contiguous Memory
```cpp
// Good - cache friendly
std::vector<Bullet> pool;  // All bullets next to each other in memory

// Bad - cache unfriendly
std::vector<Bullet*> pool;  // Bullets scattered across memory
```

### Tip 2: Early Exit in Loops
```cpp
// Good - skip inactive bullets immediately
for (int i = 0; i < poolSize; i++)
{
    if (!pool[i].IsActive()) continue;  // Skip early!
    pool[i].Update(deltaTime);
}

// Bad - checks active twice
for (int i = 0; i < poolSize; i++)
{
    if (pool[i].IsActive())
    {
        pool[i].Update(deltaTime);
    }
}
```

### Tip 3: Separate Active List (Advanced)
```cpp
// Instead of checking every bullet...
std::vector<int> activeIndices;  // List of active bullet indices

// Only update active ones
for (int index : activeIndices)
{
    pool[index].Update(deltaTime);
}
```

---

## Object Pool vs Other Patterns

### vs Factory Pattern
- **Factory:** Creates new objects each time
- **Pool:** Reuses existing objects
- **Use together:** Factory creates the pool, pool spawns objects

### vs Flyweight Pattern
- **Flyweight:** Shares data between objects (e.g., all zombies share same sprite)
- **Pool:** Manages lifetime and reuse
- **Use together:** Pooled bullets can use Flyweight for shared texture

### vs Singleton
- **Singleton:** One instance, global access
- **Pool:** Many instances, managed allocation
- **Use together:** `BulletPool::Instance()` - singleton pool!

---

## Summary

✅ **Object Pool** pre-allocates objects for reuse
✅ **Activate/Deactivate** instead of new/delete
✅ **Performance:** No allocation during gameplay
✅ **Use for:** Bullets, particles, enemies, effects
✅ **Pool size:** `fireRate × lifetime × safetyMargin`

**The pattern you just learned:**
```cpp
class ObjectPool
{
private:
    std::vector<Object> pool;
    
public:
    ObjectPool(int size) { pool.resize(size); }
    
    Object* Spawn()
    {
        // Find inactive object
        // Activate and return it
    }
};
```

**Before Pool:**
```
Create bullet → new Bullet() → 0.05ms
Destroy bullet → delete → 0.05ms
Total per bullet: 0.10ms
100 bullets/sec = 10ms wasted!
```

**After Pool:**
```
Spawn bullet → activate = true → 0.0001ms
Deactivate bullet → activate = false → 0.0001ms
Total per bullet: 0.0002ms
100 bullets/sec = 0.02ms! ✨
```

**Next Pattern:** State Pattern - organizing Menu, Playing, Paused, and GameOver states!