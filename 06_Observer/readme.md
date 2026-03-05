# Observer Pattern

## The Problem

Your **Zombie Horde** game is growing complex. When a zombie dies, multiple things need to happen:

```cpp
void Zombie::TakeDamage(int damage)
{
    health -= damage;
    
    if (health <= 0)
    {
        // When zombie dies, we need to:
        
        // 1. Add score
        GameManager::Instance().AddScore(pointValue);
        
        // 2. Play death sound
        PlaySound(zombieDeathSound);
        
        // 3. Spawn blood particles
        for (int i = 0; i < 10; i++)
        {
            particlePool.Spawn(position, GetRandomDirection(), RED);
        }
        
        // 4. Update kill counter
        GameManager::Instance().IncrementKills();
        
        // 5. Check for achievements
        if (GameManager::Instance().GetTotalKills() == 100)
        {
            AchievementManager::Instance().Unlock("CENTURION");
        }
        
        // 6. Trigger screen shake
        ScreenShake::Instance().Shake(0.2f);
        
        // 7. Update UI
        UIManager::Instance().ShowFloatingText("+10", position);
        
        isAlive = false;
    }
}
```

### What's Wrong?

**Problem 1: Tight coupling**
```cpp
// Zombie class knows about:
// - GameManager
// - ParticlePool  
// - AchievementManager
// - ScreenShake
// - UIManager
// - Audio system
// This is TERRIBLE coupling!
```

**Problem 2: Zombie class is doing too much**
```cpp
// Zombie should just... die
// Why does it care about achievements? UI? Screen shake?
```

**Problem 3: Hard to extend**
```cpp
// Want to add combo system?
// Now you need to modify Zombie::TakeDamage()
// Want to add quest tracking?
// Modify Zombie::TakeDamage() AGAIN
```

**Problem 4: Can't disable features**
```cpp
// What if we want screen shake OFF?
// Can't - it's hardcoded in the zombie!
```

**Problem 5: Testing nightmare**
```cpp
// To test zombie death, you need:
// - GameManager initialized
// - ParticlePool initialized
// - AchievementManager initialized
// - ScreenShake initialized
// - UIManager initialized
// - Audio system initialized
// Just to test if health goes to 0!
```

---

## The Solution: Observer Pattern

**Objects can "subscribe" to events and get notified when they happen.**

Think of it like **YouTube subscriptions**:
- You subscribe to a channel (event)
- When channel posts (event fires), you get notified
- You can unsubscribe anytime
- The channel doesn't know who's subscribed (decoupled!)

### Key Characteristics:
1. **Event publishers** don't know who's listening
2. **Event subscribers** can add/remove themselves
3. **Decoupled** - no direct dependencies
4. **Flexible** - add/remove listeners at runtime

---

## Implementation in Zombie Horde

### Event.h (Generic Event System)

```cpp
#ifndef EVENT_H
#define EVENT_H

#include <vector>
#include <functional>

// Generic event with data
template<typename T>
class Event
{
private:
    std::vector<std::function<void(const T&)>> listeners;
    
public:
    // Subscribe to event
    void Subscribe(std::function<void(const T&)> listener)
    {
        listeners.push_back(listener);
    }
    
    // Trigger event (notify all listeners)
    void Invoke(const T& data)
    {
        for (auto& listener : listeners)
        {
            listener(data);
        }
    }
    
    // Clear all listeners
    void Clear()
    {
        listeners.clear();
    }
};

// Event with no data
class SimpleEvent
{
private:
    std::vector<std::function<void()>> listeners;
    
public:
    void Subscribe(std::function<void()> listener)
    {
        listeners.push_back(listener);
    }
    
    void Invoke()
    {
        for (auto& listener : listeners)
        {
            listener();
        }
    }
    
    void Clear()
    {
        listeners.clear();
    }
};

#endif
```

---

### GameEvents.h (Define All Game Events)

```cpp
#ifndef GAME_EVENTS_H
#define GAME_EVENTS_H

#include "Event.h"
#include "raylib.h"

// Event data structures
struct ZombieKilledData
{
    Vector2 position;
    int pointValue;
    int zombieType;  // 0=slow, 1=fast, 2=tank
};

struct PlayerHitData
{
    int damage;
    Vector2 hitPosition;
    Vector2 attackerPosition;
};

struct WaveCompleteData
{
    int waveNumber;
    int zombiesKilled;
    float timeElapsed;
};

struct ComboData
{
    int comboCount;
    float comboMultiplier;
};

// Global event manager (Singleton)
class GameEvents
{
private:
    GameEvents() = default;
    
public:
    static GameEvents& Instance()
    {
        static GameEvents instance;
        return instance;
    }
    
    // All game events
    Event<ZombieKilledData> OnZombieKilled;
    Event<PlayerHitData> OnPlayerHit;
    Event<WaveCompleteData> OnWaveComplete;
    Event<ComboData> OnComboAchieved;
    SimpleEvent OnGameOver;
    SimpleEvent OnGameStart;
    SimpleEvent OnPause;
    SimpleEvent OnResume;
};

#endif
```

---

### Zombie.cpp (Clean - Just Fire Event!)

```cpp
#include "Zombie.h"
#include "GameEvents.h"

void Zombie::TakeDamage(int damage)
{
    health -= damage;
    
    if (health <= 0)
    {
        // Just fire the event - that's it!
        ZombieKilledData data;
        data.position = position;
        data.pointValue = pointValue;
        data.zombieType = (int)type;
        
        GameEvents::Instance().OnZombieKilled.Invoke(data);
        
        isAlive = false;
    }
}
```

**Look how clean that is!** Zombie doesn't know about score, particles, achievements, etc.

---

### ScoreSystem.cpp (Listen to Events)

```cpp
#include "ScoreSystem.h"
#include "GameEvents.h"
#include "GameManager.h"

ScoreSystem::ScoreSystem()
{
    // Subscribe to zombie killed events
    GameEvents::Instance().OnZombieKilled.Subscribe(
        [this](const ZombieKilledData& data)
        {
            // Add score when zombie dies
            int score = data.pointValue;
            GameManager::Instance().AddScore(score);
        }
    );
    
    // Subscribe to combo events (bonus points!)
    GameEvents::Instance().OnComboAchieved.Subscribe(
        [this](const ComboData& data)
        {
            int bonus = 50 * data.comboMultiplier;
            GameManager::Instance().AddScore(bonus);
        }
    );
}
```

---

### ParticleSystem.cpp (Listen to Events)

```cpp
#include "ParticleSystem.h"
#include "GameEvents.h"

ParticleSystem::ParticleSystem(ParticlePool& pool)
    : particlePool(pool)
{
    // Subscribe to zombie killed events
    GameEvents::Instance().OnZombieKilled.Subscribe(
        [this](const ZombieKilledData& data)
        {
            // Spawn blood particles
            for (int i = 0; i < 10; i++)
            {
                Vector2 randomDir = {
                    (float)(GetRandomValue(-100, 100)) / 100.0f,
                    (float)(GetRandomValue(-100, 100)) / 100.0f
                };
                
                particlePool.Spawn(data.position, randomDir, RED, 1.0f);
            }
        }
    );
    
    // Subscribe to player hit events (impact particles)
    GameEvents::Instance().OnPlayerHit.Subscribe(
        [this](const PlayerHitData& data)
        {
            // Spawn yellow impact particles
            for (int i = 0; i < 5; i++)
            {
                Vector2 randomDir = {
                    (float)(GetRandomValue(-100, 100)) / 100.0f,
                    (float)(GetRandomValue(-100, 100)) / 100.0f
                };
                
                particlePool.Spawn(data.hitPosition, randomDir, YELLOW, 0.5f);
            }
        }
    );
}
```

---

### AudioSystem.cpp (Listen to Events)

```cpp
#include "AudioSystem.h"
#include "GameEvents.h"

AudioSystem::AudioSystem()
{
    // Load sounds
    zombieDeathSound = LoadSound("zombie_death.wav");
    playerHitSound = LoadSound("player_hit.wav");
    waveCompleteSound = LoadSound("wave_complete.wav");
    comboSound = LoadSound("combo.wav");
    
    // Subscribe to events
    GameEvents::Instance().OnZombieKilled.Subscribe(
        [this](const ZombieKilledData& data)
        {
            PlaySound(zombieDeathSound);
        }
    );
    
    GameEvents::Instance().OnPlayerHit.Subscribe(
        [this](const PlayerHitData& data)
        {
            PlaySound(playerHitSound);
        }
    );
    
    GameEvents::Instance().OnWaveComplete.Subscribe(
        [this](const WaveCompleteData& data)
        {
            PlaySound(waveCompleteSound);
        }
    );
    
    GameEvents::Instance().OnComboAchieved.Subscribe(
        [this](const ComboData& data)
        {
            PlaySound(comboSound);
        }
    );
}
```

---

### AchievementSystem.cpp (Listen to Events)

```cpp
#include "AchievementSystem.h"
#include "GameEvents.h"
#include "GameManager.h"

AchievementSystem::AchievementSystem()
{
    GameEvents::Instance().OnZombieKilled.Subscribe(
        [this](const ZombieKilledData& data)
        {
            // Check kill count achievements
            int totalKills = GameManager::Instance().GetTotalKills();
            
            if (totalKills == 10 && !achievements["FIRST_BLOOD"])
            {
                UnlockAchievement("FIRST_BLOOD", "Kill 10 zombies");
            }
            else if (totalKills == 100 && !achievements["CENTURION"])
            {
                UnlockAchievement("CENTURION", "Kill 100 zombies");
            }
            else if (totalKills == 1000 && !achievements["LEGEND"])
            {
                UnlockAchievement("LEGEND", "Kill 1000 zombies");
            }
            
            // Check for boss kills
            if (data.zombieType == 4 && !achievements["BOSS_KILLER"])  // 4 = boss
            {
                UnlockAchievement("BOSS_KILLER", "Defeat a boss zombie");
            }
        }
    );
    
    GameEvents::Instance().OnComboAchieved.Subscribe(
        [this](const ComboData& data)
        {
            if (data.comboCount >= 10 && !achievements["COMBO_MASTER"])
            {
                UnlockAchievement("COMBO_MASTER", "Achieve 10x combo");
            }
        }
    );
}

void AchievementSystem::UnlockAchievement(const std::string& id, const std::string& name)
{
    achievements[id] = true;
    std::cout << "🏆 ACHIEVEMENT UNLOCKED: " << name << std::endl;
    // Show notification, save to file, etc.
}
```

---

### ScreenShake.cpp (Listen to Events)

```cpp
#include "ScreenShake.h"
#include "GameEvents.h"

ScreenShake::ScreenShake()
    : shakeAmount(0.0f)
    , shakeDuration(0.0f)
    , shakeTimer(0.0f)
{
    // Subscribe to events that cause screen shake
    GameEvents::Instance().OnPlayerHit.Subscribe(
        [this](const PlayerHitData& data)
        {
            // Shake intensity based on damage
            float intensity = data.damage / 10.0f;  // 10 damage = 1.0 intensity
            Shake(0.3f, intensity);
        }
    );
    
    GameEvents::Instance().OnZombieKilled.Subscribe(
        [this](const ZombieKilledData& data)
        {
            // Small shake on kill
            if (data.zombieType == 4)  // Boss
            {
                Shake(0.5f, 2.0f);  // Big shake!
            }
            else
            {
                Shake(0.1f, 0.3f);  // Small shake
            }
        }
    );
}

void ScreenShake::Shake(float duration, float amount)
{
    shakeDuration = duration;
    shakeAmount = amount;
    shakeTimer = 0.0f;
}

void ScreenShake::Update(float deltaTime)
{
    if (shakeTimer < shakeDuration)
    {
        shakeTimer += deltaTime;
        
        // Calculate shake offset
        float progress = shakeTimer / shakeDuration;
        float intensity = (1.0f - progress) * shakeAmount;
        
        offset.x = (GetRandomValue(-100, 100) / 100.0f) * intensity * 10.0f;
        offset.y = (GetRandomValue(-100, 100) / 100.0f) * intensity * 10.0f;
    }
    else
    {
        offset = {0, 0};
    }
}
```

---

### ComboSystem.cpp (Listen AND Fire Events!)

```cpp
#include "ComboSystem.h"
#include "GameEvents.h"

ComboSystem::ComboSystem()
    : comboCount(0)
    , comboTimer(0.0f)
    , comboWindow(2.0f)  // 2 second window
{
    // Listen to zombie kills
    GameEvents::Instance().OnZombieKilled.Subscribe(
        [this](const ZombieKilledData& data)
        {
            // Increment combo
            comboCount++;
            comboTimer = 0.0f;  // Reset timer
            
            // Fire combo event if threshold reached
            if (comboCount >= 3)
            {
                ComboData comboData;
                comboData.comboCount = comboCount;
                comboData.comboMultiplier = 1.0f + (comboCount * 0.1f);
                
                GameEvents::Instance().OnComboAchieved.Invoke(comboData);
            }
        }
    );
}

void ComboSystem::Update(float deltaTime)
{
    if (comboCount > 0)
    {
        comboTimer += deltaTime;
        
        // Reset combo if window expires
        if (comboTimer > comboWindow)
        {
            comboCount = 0;
        }
    }
}
```

---

## How to Use It

### Setup (In main.cpp or game initialization)

```cpp
int main()
{
    InitWindow(800, 600, "Zombie Horde");
    
    // Create all systems that listen to events
    ScoreSystem scoreSystem;
    ParticleSystem particleSystem(particlePool);
    AudioSystem audioSystem;
    AchievementSystem achievementSystem;
    ScreenShake screenShake;
    ComboSystem comboSystem;
    
    // That's it! Events are automatically wired up!
    
    // Game loop...
}
```

### Fire Events (From Anywhere)

```cpp
// When zombie dies:
ZombieKilledData data;
data.position = zombiePos;
data.pointValue = 10;
data.zombieType = 0;
GameEvents::Instance().OnZombieKilled.Invoke(data);

// When player is hit:
PlayerHitData hitData;
hitData.damage = 10;
hitData.hitPosition = playerPos;
hitData.attackerPosition = zombiePos;
GameEvents::Instance().OnPlayerHit.Invoke(hitData);

// When wave completes:
WaveCompleteData waveData;
waveData.waveNumber = 5;
waveData.zombiesKilled = 47;
waveData.timeElapsed = 120.5f;
GameEvents::Instance().OnWaveComplete.Invoke(waveData);

// Simple events (no data):
GameEvents::Instance().OnGameOver.Invoke();
GameEvents::Instance().OnPause.Invoke();
```

---

## How It Works

### Before Observer Pattern (Tight Coupling)

```
Zombie ────────┬────────> GameManager (add score)
               ├────────> ParticlePool (spawn particles)
               ├────────> AudioSystem (play sound)
               ├────────> AchievementSystem (check unlocks)
               ├────────> ScreenShake (shake screen)
               └────────> UIManager (floating text)

Zombie knows about EVERYTHING!
```

### After Observer Pattern (Decoupled)

```
                    OnZombieKilled Event
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ↓                   ↓                   ↓
   ScoreSystem      ParticleSystem      AudioSystem
   (subscribes)      (subscribes)       (subscribes)
        │                   │                   │
        ↓                   ↓                   ↓
   Add score         Spawn particles      Play sound
   
Zombie just fires event - doesn't know who's listening!
```

---

## Exercise 1: Basic Extension - Wave Start Event

**Goal:** Add OnWaveStart event that triggers when a new wave begins.

**Requirements:**
1. Add `OnWaveStart` event to GameEvents
2. Create `WaveStartData` struct:
    - Wave number
    - Zombie count
    - Difficulty multiplier
3. Fire the event when wave starts
4. Create listeners:
    - AudioSystem: Play dramatic music
    - UIManager: Show "Wave X" banner
    - ParticleSystem: Spawn portal effect

**Test:** Starting wave 5 should play music, show banner, and spawn particles.

---

## Exercise 2: Intermediate Challenge - Power-Up System

**Goal:** Implement a power-up system using events.

**Requirements:**
1. Add events:
    - `OnPowerUpSpawned` - power-up appears
    - `OnPowerUpCollected` - player grabs it
    - `OnPowerUpExpired` - power-up effect ends
2. Create `PowerUpData`:
    - Type (speed, health, damage)
    - Duration
    - Position
3. Systems that listen:
    - **Player:** Apply stat boost
    - **AudioSystem:** Play power-up sound
    - **UIManager:** Show icon and timer
    - **ParticleSystem:** Spawn sparkles

**Challenge:** Handle multiple active power-ups at once.

---

## Exercise 3: Advanced - Event Priority System

**Goal:** Add priority to event listeners so some execute before others.

**Problem:** Sometimes order matters!
```cpp
// Want achievements to check BEFORE score is added
// Want UI to update AFTER score changes
```

**Requirements:**
1. Modify Event class to support priority:
```cpp
void Subscribe(std::function<void(const T&)> listener, int priority = 0);
```
2. Higher priority = executes first
3. Sort listeners by priority when invoking
4. Test: Achievement listener (priority 10) should run before Score listener (priority 0)

---

## Exercise 4: Advanced (Optional) - Event History & Replay

**Goal:** Record all events and replay them (for debugging or replay system).

**Requirements:**
1. Create `EventRecorder` class
2. Records all events with timestamp
3. Can save to file / load from file
4. Can replay events at normal speed or fast-forward
5. Shows event name, data, and timestamp

**Use cases:**
- Debug: "Why did player die? Let me replay the last 10 seconds"
- Replay system: Record gameplay and play it back
- Testing: Record a bug scenario and replay it

**Example output:**
```
[0.00s] OnGameStart
[0.53s] OnZombieKilled { pos: (100, 200), points: 10 }
[0.89s] OnZombieKilled { pos: (150, 180), points: 10 }
[1.20s] OnComboAchieved { count: 2, multiplier: 1.2 }
[2.45s] OnPlayerHit { damage: 10, pos: (400, 300) }
```

---

## Common Mistakes

### ❌ Mistake 1: Forgetting to subscribe

```cpp
// Create system but forget to subscribe
AudioSystem audioSystem;  // Sounds never play!

// In AudioSystem constructor, forgot:
// GameEvents::Instance().OnZombieKilled.Subscribe(...);
```

**Fix:** Subscribe in constructor
```cpp
AudioSystem::AudioSystem()
{
    GameEvents::Instance().OnZombieKilled.Subscribe([this](auto& data) {
        PlaySound(zombieDeathSound);
    });
}
```

---

### ❌ Mistake 2: Memory leaks with lambdas capturing `this`

```cpp
// WRONG - listener keeps object alive forever
void SomeSystem::Setup()
{
    GameEvents::Instance().OnZombieKilled.Subscribe(
        [this](const ZombieKilledData& data) {
            this->DoSomething();  // 'this' is captured!
        }
    );
}

// When SomeSystem is deleted, the lambda still holds a pointer!
// Event tries to call deleted object → CRASH
```

**Fix:** Unsubscribe in destructor (advanced) or use weak_ptr
```cpp
// Simple fix: Clear events on cleanup
SomeSystem::~SomeSystem()
{
    // If this is the only listener, clear it
    // (Better: add Unsubscribe functionality)
}
```

---

### ❌ Mistake 3: Infinite event loops

```cpp
// WRONG - event triggers itself!
GameEvents::Instance().OnScoreChanged.Subscribe([](int score) {
    GameManager::Instance().AddScore(10);  // This fires OnScoreChanged!
    // Infinite loop!
});
```

**Fix:** Be careful what you do in listeners
```cpp
// CORRECT - don't trigger same event
GameEvents::Instance().OnScoreChanged.Subscribe([](int score) {
    // Just display the new score, don't modify it
    UpdateScoreUI(score);
});
```

---

### ❌ Mistake 4: Passing references to temporary data

```cpp
// WRONG - data is destroyed after Invoke() returns
void Zombie::Die()
{
    ZombieKilledData data;
    data.position = position;
    GameEvents::Instance().OnZombieKilled.Invoke(data);
    // 'data' is destroyed here
}

// Listener tries to use it later
GameEvents::Instance().OnZombieKilled.Subscribe([](const ZombieKilledData& data) {
    // If this runs asynchronously, data might be invalid!
    SpawnParticlesLater(data.position);  // DANGER
});
```

**Fix:** Copy data if needed asynchronously
```cpp
GameEvents::Instance().OnZombieKilled.Subscribe([](const ZombieKilledData& data) {
    Vector2 pos = data.position;  // Copy what you need
    SpawnParticlesLater(pos);
});
```

---

### ❌ Mistake 5: Too many events

```cpp
// WRONG - event for everything!
OnZombieSpawned
OnZombieMoved
OnZombieRotated
OnZombieStopped
OnZombieStartedMoving
// ... 50 more events
```

**Fix:** Only create events for important moments
```cpp
// CORRECT - key moments only
OnZombieKilled
OnPlayerHit
OnWaveComplete
OnGameOver
```

**Rule:** If it happens every frame, it's probably not an event.

---

## When to Use Observer Pattern

### ✅ Good Use Cases:
- **Game events** (death, level up, achievement)
- **UI updates** (score changed, health changed)
- **Achievement systems** (triggered by various events)
- **Audio triggers** (play sound on event)
- **Analytics** (track player actions)
- **Combo systems** (chaining actions)

### ❌ Bad Use Cases:
- **Per-frame updates** (use normal Update() calls)
- **Direct communication** (if A always talks to B, just call B directly)
- **Two-way communication** (events are one-way)
- **When you need return values** (events can't return data easily)

### The Rule of Thumb:
**"If multiple unrelated systems care about the same moment, use an event."**

---

## Observer Pattern Variations

### 1. Function-based (What we built)
```cpp
Event<DataType> OnSomething;
OnSomething.Subscribe([](const DataType& data) { /* ... */ });
OnSomething.Invoke(data);
```
- Simple, flexible
- Uses std::function and lambdas

### 2. Interface-based
```cpp
class IEventListener
{
    virtual void OnEvent(const DataType& data) = 0;
};

class MySystem : public IEventListener
{
    void OnEvent(const DataType& data) override { /* ... */ }
};
```
- More traditional OOP
- Type-safe

### 3. Delegate-based (C# style)
```cpp
class Event
{
    void operator+=(std::function<void()> listener);  // Subscribe
    void operator-=(std::function<void()> listener);  // Unsubscribe
    void operator()();  // Invoke
};
```
- Nice syntax: `OnEvent += handler;`

---

## Summary

✅ **Observer Pattern** decouples event publishers from subscribers<br>
✅ **Subscribe** to events you care about<br>
✅ **Invoke** events when moments happen<br>
✅ **Flexible** - add/remove listeners dynamically<br>
✅ **Clean code** - zombie just fires event, doesn't know about systems

**The pattern you just learned:**
```cpp
// 1. Define event
Event<DataType> OnSomething;

// 2. Subscribe
OnSomething.Subscribe([](const DataType& data) {
    // React to event
});

// 3. Fire event
DataType data = { /* ... */ };
OnSomething.Invoke(data);
```

**Before Observer:**
```cpp
void Zombie::Die()
{
    GameManager::AddScore(10);
    ParticlePool::Spawn(...);
    AudioSystem::Play(...);
    AchievementSystem::Check(...);
    ScreenShake::Shake(...);
    UIManager::ShowText(...);
    // Zombie knows about EVERYTHING! 😱
}
```

**After Observer:**
```cpp
void Zombie::Die()
{
    ZombieKilledData data = { position, points, type };
    GameEvents::Instance().OnZombieKilled.Invoke(data);
    // Clean! Zombie just reports what happened. ✨
}
```

**Next Pattern:** Component Pattern - building flexible entities with composition!