# Strategy Pattern

## The Problem

Your **Zombie Horde** needs different AI behaviors. You start with a giant if/else in the zombie's update:

```cpp
class Zombie
{
private:
    enum AIState { CHASE, FLEE, WANDER, CIRCLE };
    AIState currentState;
    
public:
    void Update(float deltaTime, Vector2 playerPos)
    {
        if (currentState == CHASE)
        {
            // Calculate direction to player
            Vector2 direction = Normalize(playerPos - position);
            velocity = direction * chaseSpeed;
            position += velocity * deltaTime;
        }
        else if (currentState == FLEE)
        {
            // Calculate direction away from player
            Vector2 direction = Normalize(position - playerPos);
            velocity = direction * fleeSpeed;
            position += velocity * deltaTime;
        }
        else if (currentState == WANDER)
        {
            // Random movement
            if (rand() % 100 < 2)  // 2% chance to change direction
            {
                wanderDirection = GetRandomDirection();
            }
            velocity = wanderDirection * wanderSpeed;
            position += velocity * deltaTime;
        }
        else if (currentState == CIRCLE)
        {
            // Move in circle around player
            Vector2 toPlayer = playerPos - position;
            float angle = atan2(toPlayer.y, toPlayer.x);
            angle += circleSpeed * deltaTime;
            
            float radius = 100.0f;
            Vector2 target = {
                playerPos.x + cos(angle) * radius,
                playerPos.y + sin(angle) * radius
            };
            
            Vector2 direction = Normalize(target - position);
            velocity = direction * moveSpeed;
            position += velocity * deltaTime;
        }
        
        // Check health to decide state
        if (health < maxHealth * 0.3f)
        {
            currentState = FLEE;  // Low health - run away!
        }
        else if (DistanceToPlayer(playerPos) > 200.0f)
        {
            currentState = WANDER;  // Too far - wander
        }
        else if (DistanceToPlayer(playerPos) < 50.0f)
        {
            currentState = CHASE;  // Close - chase!
        }
    }
};
```

### What's Wrong?

**Problem 1: Giant Update() method**
```cpp
// 100+ lines of if/else statements
// Hard to read
// Hard to debug
// "Which state is broken?"
```

**Problem 2: Can't reuse behaviors**
```cpp
// Want player to also use FLEE behavior?
// Copy-paste the entire flee code!
// Now you have duplicate code in Player and Zombie
```

**Problem 3: Hard to add new behaviors**
```cpp
// Add AMBUSH behavior?
// - Add to enum
// - Add another if/else branch
// - Modify the Update() method
// - Now Update() is 150 lines long!
```

**Problem 4: Can't change behavior at runtime easily**
```cpp
// Want zombie to use different strategies per type?
// FastZombie uses CHASE + CIRCLE
// TankZombie uses CHASE only
// Need different zombie classes? More inheritance?
```

**Problem 5: Testing nightmare**
```cpp
// To test FLEE behavior:
// - Set zombie health to 20%
// - Set player position
// - Hope the if/else picks FLEE state
// - Can't test FLEE in isolation!
```

---

## The Solution: Strategy Pattern

**Encapsulate each behavior in its own class, swap them at runtime.**

Think of it like **Pokemon moves**:
- Pokemon has 4 move slots
- Each move is a different strategy
- Swap moves in/out between battles
- Same Pokemon, different strategies

### Key Characteristics:
1. **Each behavior is a class** (ChaseStrategy, FleeStrategy, etc.)
2. **Common interface** (all strategies have Execute())
3. **Swappable at runtime** - change strategy any time
4. **Testable** - test each strategy independently

---

## Implementation in Zombie Horde

### MovementStrategy.h (Interface)

```cpp
#ifndef MOVEMENT_STRATEGY_H
#define MOVEMENT_STRATEGY_H

#include "raylib.h"

// Base class for all movement strategies
class MovementStrategy
{
public:
    virtual ~MovementStrategy() = default;
    
    // Calculate desired velocity based on current state
    virtual Vector2 CalculateVelocity(
        Vector2 currentPosition,
        Vector2 targetPosition,
        float deltaTime
    ) = 0;
    
    // Optional: called when strategy starts
    virtual void OnStart() {}
    
    // Optional: called when strategy ends
    virtual void OnEnd() {}
    
    // Get strategy name (for debugging)
    virtual const char* GetName() const = 0;
};

#endif
```

---

### ChaseStrategy.h

```cpp
#ifndef CHASE_STRATEGY_H
#define CHASE_STRATEGY_H

#include "MovementStrategy.h"
#include <cmath>

class ChaseStrategy : public MovementStrategy
{
private:
    float speed;
    
public:
    ChaseStrategy(float chaseSpeed = 100.0f)
        : speed(chaseSpeed)
    {}
    
    Vector2 CalculateVelocity(
        Vector2 currentPosition,
        Vector2 targetPosition,
        float deltaTime
    ) override
    {
        // Calculate direction to target
        Vector2 direction = {
            targetPosition.x - currentPosition.x,
            targetPosition.y - currentPosition.y
        };
        
        // Normalize
        float length = sqrtf(direction.x * direction.x + direction.y * direction.y);
        if (length > 0.001f)
        {
            direction.x /= length;
            direction.y /= length;
        }
        
        // Return velocity
        return {
            direction.x * speed,
            direction.y * speed
        };
    }
    
    void SetSpeed(float spd) { speed = spd; }
    float GetSpeed() const { return speed; }
    
    const char* GetName() const override { return "Chase"; }
};

#endif
```

---

### FleeStrategy.h

```cpp
#ifndef FLEE_STRATEGY_H
#define FLEE_STRATEGY_H

#include "MovementStrategy.h"
#include <cmath>

class FleeStrategy : public MovementStrategy
{
private:
    float speed;
    float safeDistance;  // How far to run before stopping
    
public:
    FleeStrategy(float fleeSpeed = 150.0f, float safeDist = 300.0f)
        : speed(fleeSpeed)
        , safeDistance(safeDist)
    {}
    
    Vector2 CalculateVelocity(
        Vector2 currentPosition,
        Vector2 targetPosition,
        float deltaTime
    ) override
    {
        // Calculate direction AWAY from target
        Vector2 direction = {
            currentPosition.x - targetPosition.x,
            currentPosition.y - targetPosition.y
        };
        
        // Calculate distance
        float distance = sqrtf(direction.x * direction.x + direction.y * direction.y);
        
        // If far enough, stop fleeing
        if (distance >= safeDistance)
        {
            return {0, 0};
        }
        
        // Normalize
        if (distance > 0.001f)
        {
            direction.x /= distance;
            direction.y /= distance;
        }
        
        // Return velocity away from target
        return {
            direction.x * speed,
            direction.y * speed
        };
    }
    
    void SetSpeed(float spd) { speed = spd; }
    void SetSafeDistance(float dist) { safeDistance = dist; }
    
    const char* GetName() const override { return "Flee"; }
};

#endif
```

---

### WanderStrategy.h

```cpp
#ifndef WANDER_STRATEGY_H
#define WANDER_STRATEGY_H

#include "MovementStrategy.h"
#include "raylib.h"
#include <cmath>

class WanderStrategy : public MovementStrategy
{
private:
    float speed;
    Vector2 wanderDirection;
    float changeDirectionTimer;
    float changeDirectionInterval;
    
public:
    WanderStrategy(float wanderSpeed = 50.0f)
        : speed(wanderSpeed)
        , wanderDirection({1, 0})
        , changeDirectionTimer(0.0f)
        , changeDirectionInterval(2.0f)
    {}
    
    void OnStart() override
    {
        // Pick random direction when starting
        PickNewDirection();
    }
    
    Vector2 CalculateVelocity(
        Vector2 currentPosition,
        Vector2 targetPosition,
        float deltaTime
    ) override
    {
        // Update timer
        changeDirectionTimer += deltaTime;
        
        // Change direction periodically
        if (changeDirectionTimer >= changeDirectionInterval)
        {
            changeDirectionTimer = 0.0f;
            PickNewDirection();
        }
        
        return {
            wanderDirection.x * speed,
            wanderDirection.y * speed
        };
    }
    
    void PickNewDirection()
    {
        // Random angle
        float angle = (GetRandomValue(0, 360) * PI) / 180.0f;
        wanderDirection = {
            cosf(angle),
            sinf(angle)
        };
    }
    
    void SetSpeed(float spd) { speed = spd; }
    void SetChangeInterval(float interval) { changeDirectionInterval = interval; }
    
    const char* GetName() const override { return "Wander"; }
};

#endif
```

---

### CircleStrategy.h

```cpp
#ifndef CIRCLE_STRATEGY_H
#define CIRCLE_STRATEGY_H

#include "MovementStrategy.h"
#include <cmath>

class CircleStrategy : public MovementStrategy
{
private:
    float speed;
    float radius;
    float currentAngle;
    bool clockwise;
    
public:
    CircleStrategy(float circleSpeed = 80.0f, float circleRadius = 150.0f, bool isClockwise = true)
        : speed(circleSpeed)
        , radius(circleRadius)
        , currentAngle(0.0f)
        , clockwise(isClockwise)
    {}
    
    void OnStart() override
    {
        // Calculate starting angle based on current position
        currentAngle = 0.0f;
    }
    
    Vector2 CalculateVelocity(
        Vector2 currentPosition,
        Vector2 targetPosition,
        float deltaTime
    ) override
    {
        // Calculate current angle relative to target
        Vector2 toTarget = {
            currentPosition.x - targetPosition.x,
            currentPosition.y - targetPosition.y
        };
        currentAngle = atan2f(toTarget.y, toTarget.x);
        
        // Increment angle (circle around target)
        float angleChange = (speed / radius) * deltaTime;
        if (!clockwise)
        {
            angleChange = -angleChange;
        }
        currentAngle += angleChange;
        
        // Calculate desired position on circle
        Vector2 desiredPosition = {
            targetPosition.x + cosf(currentAngle) * radius,
            targetPosition.y + sinf(currentAngle) * radius
        };
        
        // Calculate direction to desired position
        Vector2 direction = {
            desiredPosition.x - currentPosition.x,
            desiredPosition.y - currentPosition.y
        };
        
        // Normalize
        float length = sqrtf(direction.x * direction.x + direction.y * direction.y);
        if (length > 0.001f)
        {
            direction.x /= length;
            direction.y /= length;
        }
        
        // Move toward desired circle position
        return {
            direction.x * speed,
            direction.y * speed
        };
    }
    
    void SetSpeed(float spd) { speed = spd; }
    void SetRadius(float r) { radius = r; }
    void SetClockwise(bool cw) { clockwise = cw; }
    
    const char* GetName() const override { return "Circle"; }
};

#endif
```

---

### ZigZagStrategy.h

```cpp
#ifndef ZIGZAG_STRATEGY_H
#define ZIGZAG_STRATEGY_H

#include "MovementStrategy.h"
#include <cmath>

class ZigZagStrategy : public MovementStrategy
{
private:
    float speed;
    float zigzagAmount;
    float zigzagFrequency;
    float timer;
    
public:
    ZigZagStrategy(float moveSpeed = 100.0f, float amount = 50.0f, float frequency = 2.0f)
        : speed(moveSpeed)
        , zigzagAmount(amount)
        , zigzagFrequency(frequency)
        , timer(0.0f)
    {}
    
    Vector2 CalculateVelocity(
        Vector2 currentPosition,
        Vector2 targetPosition,
        float deltaTime
    ) override
    {
        timer += deltaTime;
        
        // Calculate base direction to target
        Vector2 direction = {
            targetPosition.x - currentPosition.x,
            targetPosition.y - currentPosition.y
        };
        
        float length = sqrtf(direction.x * direction.x + direction.y * direction.y);
        if (length > 0.001f)
        {
            direction.x /= length;
            direction.y /= length;
        }
        
        // Calculate perpendicular direction for zigzag
        Vector2 perpendicular = {
            -direction.y,
            direction.x
        };
        
        // Add sine wave offset
        float offset = sinf(timer * zigzagFrequency) * zigzagAmount;
        
        Vector2 velocity = {
            direction.x * speed + perpendicular.x * offset,
            direction.y * speed + perpendicular.y * offset
        };
        
        return velocity;
    }
    
    void SetSpeed(float spd) { speed = spd; }
    void SetZigzagAmount(float amount) { zigzagAmount = amount; }
    void SetZigzagFrequency(float freq) { zigzagFrequency = freq; }
    
    const char* GetName() const override { return "ZigZag"; }
};

#endif
```

---

### AI.h (Uses Strategy)

```cpp
#ifndef AI_H
#define AI_H

#include "MovementStrategy.h"
#include "raylib.h"
#include <memory>

class AI
{
private:
    std::unique_ptr<MovementStrategy> currentStrategy;
    Vector2 position;
    Vector2 velocity;
    
public:
    AI(Vector2 startPos)
        : position(startPos)
        , velocity({0, 0})
        , currentStrategy(nullptr)
    {}
    
    // Set strategy
    void SetStrategy(MovementStrategy* strategy)
    {
        // Call OnEnd() on old strategy
        if (currentStrategy)
        {
            currentStrategy->OnEnd();
        }
        
        // Set new strategy
        currentStrategy.reset(strategy);
        
        // Call OnStart() on new strategy
        if (currentStrategy)
        {
            currentStrategy->OnStart();
        }
    }
    
    // Update AI
    void Update(Vector2 targetPosition, float deltaTime)
    {
        if (!currentStrategy) return;
        
        // Get velocity from strategy
        velocity = currentStrategy->CalculateVelocity(position, targetPosition, deltaTime);
        
        // Apply velocity
        position.x += velocity.x * deltaTime;
        position.y += velocity.y * deltaTime;
    }
    
    // Getters
    Vector2 GetPosition() const { return position; }
    Vector2 GetVelocity() const { return velocity; }
    const char* GetCurrentStrategyName() const
    {
        return currentStrategy ? currentStrategy->GetName() : "None";
    }
};

#endif
```

---

## How to Use It

### Simple Usage - Single Strategy

```cpp
// Create AI
AI zombieAI({100, 100});

// Set chase strategy
zombieAI.SetStrategy(new ChaseStrategy(80.0f));

// Update every frame
zombieAI.Update(playerPosition, deltaTime);
```

### Switching Strategies Based on Conditions

```cpp
class SmartZombie
{
private:
    AI ai;
    int health;
    int maxHealth;
    
public:
    void Update(Vector2 playerPos, float deltaTime)
    {
        float distance = CalculateDistance(ai.GetPosition(), playerPos);
        float healthPercent = (float)health / (float)maxHealth;
        
        // Choose strategy based on health and distance
        if (healthPercent < 0.3f)
        {
            // Low health - flee!
            ai.SetStrategy(new FleeStrategy(150.0f));
        }
        else if (distance < 50.0f)
        {
            // Very close - circle around player
            ai.SetStrategy(new CircleStrategy(100.0f, 80.0f));
        }
        else if (distance < 200.0f)
        {
            // Medium distance - chase
            ai.SetStrategy(new ChaseStrategy(80.0f));
        }
        else
        {
            // Far away - wander
            ai.SetStrategy(new WanderStrategy(40.0f));
        }
        
        // Update AI with chosen strategy
        ai.Update(playerPos, deltaTime);
    }
};
```

### Different Zombie Types with Different Strategies

```cpp
// Fast zombie - aggressive zigzag chase
Entity* fastZombie = CreateZombie(ZombieType::FAST);
fastZombie->GetAI()->SetStrategy(new ZigZagStrategy(120.0f, 30.0f, 3.0f));

// Tank zombie - slow but relentless chase
Entity* tankZombie = CreateZombie(ZombieType::TANK);
tankZombie->GetAI()->SetStrategy(new ChaseStrategy(40.0f));

// Scout zombie - circles around player
Entity* scoutZombie = CreateZombie(ZombieType::SCOUT);
scoutZombie->GetAI()->SetStrategy(new CircleStrategy(90.0f, 200.0f));

// Coward zombie - keeps distance
Entity* cowardZombie = CreateZombie(ZombieType::COWARD);
cowardZombie->GetAI()->SetStrategy(new FleeStrategy(100.0f, 250.0f));
```

---

## How It Works

### Before Strategy Pattern (Monolithic)

```
class Zombie
{
    void Update()
    {
        if (state == CHASE) { /* 20 lines */ }
        else if (state == FLEE) { /* 20 lines */ }
        else if (state == WANDER) { /* 20 lines */ }
        else if (state == CIRCLE) { /* 30 lines */ }
        else if (state == ZIGZAG) { /* 25 lines */ }
        // 115 lines total!
    }
}
```

### After Strategy Pattern (Modular)

```
class Zombie
{
    MovementStrategy* strategy;
    
    void Update()
    {
        velocity = strategy->CalculateVelocity(...);
        // 1 line!
    }
}

┌─────────────────┐
│ ChaseStrategy   │ → 20 lines (isolated)
├─────────────────┤
│ FleeStrategy    │ → 20 lines (isolated)
├─────────────────┤
│ WanderStrategy  │ → 20 lines (isolated)
├─────────────────┤
│ CircleStrategy  │ → 30 lines (isolated)
├─────────────────┤
│ ZigZagStrategy  │ → 25 lines (isolated)
└─────────────────┘
Each strategy is separate, testable, reusable!
```

---

## Exercise 1: Basic Extension - Patrol Strategy

**Goal:** Create a PatrolStrategy that moves between waypoints.

**Requirements:**
1. Create `PatrolStrategy` class
2. Store array of waypoints (Vector2)
3. Move toward current waypoint
4. When reached, move to next waypoint
5. Loop back to first waypoint after last

**Starter code:**
```cpp
class PatrolStrategy : public MovementStrategy
{
private:
    std::vector<Vector2> waypoints;
    int currentWaypointIndex;
    float speed;
    float reachedDistance;  // How close to consider "reached"
    
public:
    PatrolStrategy(float patrolSpeed = 60.0f)
        : currentWaypointIndex(0)
        , speed(patrolSpeed)
        , reachedDistance(10.0f)
    {}
    
    void AddWaypoint(Vector2 point)
    {
        waypoints.push_back(point);
    }
    
    Vector2 CalculateVelocity(
        Vector2 currentPosition,
        Vector2 targetPosition,  // Unused - we use waypoints instead
        float deltaTime
    ) override
    {
        if (waypoints.empty()) return {0, 0};
        
        // Get current waypoint
        Vector2 target = waypoints[currentWaypointIndex];
        
        // Calculate distance to waypoint
        float dx = target.x - currentPosition.x;
        float dy = target.y - currentPosition.y;
        float distance = sqrtf(dx * dx + dy * dy);
        
        // Check if reached
        if (distance < reachedDistance)
        {
            // Move to next waypoint
            currentWaypointIndex++;
            if (currentWaypointIndex >= waypoints.size())
            {
                currentWaypointIndex = 0;  // Loop back
            }
            
            // Get new target
            target = waypoints[currentWaypointIndex];
            dx = target.x - currentPosition.x;
            dy = target.y - currentPosition.y;
            distance = sqrtf(dx * dx + dy * dy);
        }
        
        // Calculate velocity toward waypoint
        if (distance > 0.001f)
        {
            return {
                (dx / distance) * speed,
                (dy / distance) * speed
            };
        }
        
        return {0, 0};
    }
    
    const char* GetName() const override { return "Patrol"; }
};
```

**Test:** Create zombie that patrols between 4 corners of screen.

---

## Exercise 2: Intermediate Challenge - Composite Strategy

**Goal:** Combine multiple strategies (e.g., Chase + Wander).

**Scenario:** Zombie chases player, but adds small wander offsets for natural movement.

**Requirements:**
1. Create `CompositeStrategy` class
2. Store two strategies: primary and secondary
3. CalculateVelocity returns: `primary + secondary * weight`
4. Weight controls how much secondary influences movement (0.0 to 1.0)

**Starter code:**
```cpp
class CompositeStrategy : public MovementStrategy
{
private:
    std::unique_ptr<MovementStrategy> primaryStrategy;
    std::unique_ptr<MovementStrategy> secondaryStrategy;
    float secondaryWeight;
    
public:
    CompositeStrategy(
        MovementStrategy* primary,
        MovementStrategy* secondary,
        float weight = 0.2f
    )
        : primaryStrategy(primary)
        , secondaryStrategy(secondary)
        , secondaryWeight(weight)
    {}
    
    Vector2 CalculateVelocity(
        Vector2 currentPosition,
        Vector2 targetPosition,
        float deltaTime
    ) override
    {
        Vector2 primaryVel = primaryStrategy->CalculateVelocity(
            currentPosition, targetPosition, deltaTime
        );
        
        Vector2 secondaryVel = secondaryStrategy->CalculateVelocity(
            currentPosition, targetPosition, deltaTime
        );
        
        // Blend velocities
        return {
            primaryVel.x + secondaryVel.x * secondaryWeight,
            primaryVel.y + secondaryVel.y * secondaryWeight
        };
    }
    
    const char* GetName() const override { return "Composite"; }
};

// Usage:
auto* chaseWithWander = new CompositeStrategy(
    new ChaseStrategy(100.0f),   // Primary: chase
    new WanderStrategy(30.0f),   // Secondary: wander
    0.3f                         // 30% wander influence
);
zombieAI.SetStrategy(chaseWithWander);
```

---

## Exercise 3: Advanced - State Machine Strategy

**Goal:** Strategy that switches between sub-strategies based on conditions.

**Requirements:**
1. Create `StateMachineStrategy`
2. Store multiple strategies in a map
3. Check conditions each frame
4. Switch to appropriate strategy
5. Example: "Chase if close, Flee if low health, Wander otherwise"

**Starter code:**
```cpp
class StateMachineStrategy : public MovementStrategy
{
private:
    struct StrategyState
    {
        MovementStrategy* strategy;
        std::function<bool(Vector2, Vector2, int, int)> condition;
        // condition(myPos, targetPos, health, maxHealth) -> should activate?
    };
    
    std::vector<StrategyState> states;
    MovementStrategy* currentStrategy;
    int health;
    int maxHealth;
    
public:
    void AddState(
        MovementStrategy* strategy,
        std::function<bool(Vector2, Vector2, int, int)> condition
    )
    {
        states.push_back({strategy, condition});
    }
    
    void SetHealth(int hp, int maxHp)
    {
        health = hp;
        maxHealth = maxHp;
    }
    
    Vector2 CalculateVelocity(
        Vector2 currentPosition,
        Vector2 targetPosition,
        float deltaTime
    ) override
    {
        // Check conditions and pick strategy
        for (auto& state : states)
        {
            if (state.condition(currentPosition, targetPosition, health, maxHealth))
            {
                currentStrategy = state.strategy;
                break;
            }
        }
        
        // Use current strategy
        if (currentStrategy)
        {
            return currentStrategy->CalculateVelocity(
                currentPosition, targetPosition, deltaTime
            );
        }
        
        return {0, 0};
    }
    
    const char* GetName() const override
    {
        return currentStrategy ? currentStrategy->GetName() : "StateMachine";
    }
};

// Usage:
auto* smartAI = new StateMachineStrategy();

// Flee if low health
smartAI->AddState(
    new FleeStrategy(150.0f),
    [](Vector2 myPos, Vector2 target, int hp, int maxHp) {
        return (float)hp / maxHp < 0.3f;  // < 30% health
    }
);

// Chase if close
smartAI->AddState(
    new ChaseStrategy(100.0f),
    [](Vector2 myPos, Vector2 target, int hp, int maxHp) {
        float dx = target.x - myPos.x;
        float dy = target.y - myPos.y;
        float dist = sqrtf(dx*dx + dy*dy);
        return dist < 200.0f;
    }
);

// Wander otherwise (default - always true)
smartAI->AddState(
    new WanderStrategy(50.0f),
    [](Vector2 myPos, Vector2 target, int hp, int maxHp) {
        return true;  // Default
    }
);
```

---

## Exercise 4: Advanced (Optional) - Learning AI

**Goal:** AI that adapts its strategy based on success/failure.

**Concept:**
- Track how effective each strategy is
- Measure: "damage dealt vs damage taken"
- Use the strategy that performs best

**Requirements:**
1. Track performance metrics for each strategy
2. Switch to best-performing strategy over time
3. Reset metrics each wave

**This is advanced ML territory - just brainstorm the design!**

---

## Common Mistakes

### ❌ Mistake 1: Not calling OnStart/OnEnd

```cpp
// WRONG - directly swapping strategies
currentStrategy = new ChaseStrategy();  // Old strategy not cleaned up!
```

**Fix:** Use SetStrategy method
```cpp
// CORRECT
ai.SetStrategy(new ChaseStrategy());  // Calls OnEnd/OnStart properly
```

---

### ❌ Mistake 2: Strategy depends on entity state

```cpp
// WRONG - strategy accessing entity's health
class FleeWhenHurtStrategy : public MovementStrategy
{
    Zombie* zombie;  // BAD! Strategy shouldn't know about zombie!
    
    Vector2 CalculateVelocity(...)
    {
        if (zombie->GetHealth() < 30)  // Tight coupling!
        {
            // Flee
        }
    }
};
```

**Fix:** Pass data as parameters or use callbacks
```cpp
// CORRECT - strategy is pure, entity decides when to flee
if (zombie.GetHealth() < 30)
{
    zombie.SetStrategy(new FleeStrategy());
}
```

---

### ❌ Mistake 3: Creating new strategy every frame

```cpp
// WRONG - memory leak!
void Update()
{
    ai.SetStrategy(new ChaseStrategy());  // Creates new object every frame!
}
```

**Fix:** Only change strategy when needed
```cpp
// CORRECT - check if change is needed
if (needToChangeStrategy)
{
    ai.SetStrategy(new ChaseStrategy());
}
```

---

### ❌ Mistake 4: Strategies that don't return velocity

```cpp
// WRONG - strategy moves entity directly
class BadStrategy : public MovementStrategy
{
    Vector2 CalculateVelocity(...) override
    {
        // DON'T DO THIS in the strategy!
        entity->position.x += 10;
        entity->position.y += 10;
        
        return {0, 0};  // Returns nothing useful
    }
};
```

**Fix:** Return velocity, let entity apply it
```cpp
// CORRECT - return the desired velocity
Vector2 CalculateVelocity(...) override
{
    return {100, 100};  // Entity will apply this
}
```

---

### ❌ Mistake 5: Over-engineering

```cpp
// WRONG - too many strategies!
ChaseStrategy
ChaseSlowStrategy
ChaseFastStrategy
ChaseZigZagStrategy
ChaseCircleStrategy
ChaseFleeStrategy
// ... 50 more strategies
```

**Fix:** Use parameterized strategies
```cpp
// CORRECT - one flexible strategy
ChaseStrategy chase(80.0f);    // Regular chase
ChaseStrategy fastChase(150.0f);  // Fast chase (same class!)
```

---

## When to Use Strategy Pattern

### ✅ Good Use Cases:
- **AI behaviors** (chase, flee, patrol, wander)
- **Pathfinding algorithms** (A*, Dijkstra, BFS)
- **Difficulty settings** (easy/normal/hard AI)
- **Attack patterns** (melee, ranged, magic)
- **Sorting algorithms** (quicksort, mergesort, bubblesort)

### ❌ Bad Use Cases:
- **Only one behavior** (just write a method!)
- **Behaviors never change** (hardcode it)
- **Very simple logic** (if/else is fine)

### The Rule of Thumb:
**"If you have 3+ algorithms for the same task that need to be swappable, use Strategy."**

---

## Summary

✅ **Strategy Pattern** encapsulates algorithms in classes<br>
✅ **Swappable** - change behavior at runtime<br>
✅ **Testable** - test each strategy independently<br>
✅ **Reusable** - same strategy for player, enemies, NPCs<br>
✅ **Clean code** - no giant if/else chains

**The pattern you just learned:**
```cpp
// 1. Define interface
class Strategy
{
    virtual Result Execute(...) = 0;
};

// 2. Create concrete strategies
class ConcreteStrategyA : public Strategy { /* ... */ };
class ConcreteStrategyB : public Strategy { /* ... */ };

// 3. Use strategy
entity.SetStrategy(new ConcreteStrategyA());
entity.Update();  // Uses strategy A

entity.SetStrategy(new ConcreteStrategyB());
entity.Update();  // Uses strategy B
```

**Before Strategy:**
```cpp
void Zombie::Update()
{
    if (state == CHASE) { /* 20 lines */ }
    else if (state == FLEE) { /* 20 lines */ }
    else if (state == WANDER) { /* 20 lines */ }
    else if (state == CIRCLE) { /* 30 lines */ }
    // 90 lines in one method! 😱
}
```

**After Strategy:**
```cpp
void Zombie::Update()
{
    velocity = strategy->CalculateVelocity(position, target, deltaTime);
    // Clean! Strategy handles the how. ✨
}

// Strategies:
FastZombie: ZigZagStrategy
TankZombie: ChaseStrategy
ScoutZombie: CircleStrategy
CowardZombie: FleeStrategy
```

**Next Pattern:** Command Pattern - the final pattern! Input handling, undo/redo, and replay systems!