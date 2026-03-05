# Component Pattern

## The Problem

Your **Zombie Horde** game has multiple entity types. You start with inheritance:

```cpp
class Entity
{
protected:
    Vector2 position;
    Vector2 velocity;
    float health;
    bool isAlive;
    
public:
    virtual void Update(float deltaTime) = 0;
    virtual void Draw() const = 0;
};

class Player : public Entity
{
private:
    float speed;
    int ammo;
    float shootCooldown;
    // Player-specific stuff
};

class Zombie : public Entity
{
private:
    float speed;
    int damage;
    ZombieType type;
    // Zombie-specific stuff
};

class Bullet : public Entity
{
private:
    float speed;
    int damage;
    // Bullet-specific stuff
};
```

Now you want to add a **health bar** to zombies. Easy, right?

```cpp
class Zombie : public Entity
{
    void DrawHealthBar();  // Add health bar
};
```

But wait, the **player also needs a health bar**! Copy-paste the code?

```cpp
class Player : public Entity
{
    void DrawHealthBar();  // Duplicate code!
};
```

Now you want **explosive barrels** that can be destroyed:

```cpp
class Barrel : public Entity
{
    // Barrels have health but don't move...
    // But Entity forces us to have velocity!
};
```

And **power-ups** that move but have no health:

```cpp
class PowerUp : public Entity
{
    // PowerUps move but have no health...
    // But Entity forces us to have health!
};
```

### What's Wrong?

**Problem 1: Rigid hierarchy**
```cpp
// Entity forces ALL subclasses to have:
// - position
// - velocity
// - health
// Not every entity needs all of these!
```

**Problem 2: Code duplication**
```cpp
// DrawHealthBar() duplicated in Player and Zombie
// Movement code duplicated everywhere
// Collision code duplicated everywhere
```

**Problem 3: Can't mix behaviors easily**
```cpp
// Want a zombie that explodes? 
// Can't inherit from both Zombie AND Exploder
// C++ doesn't support multiple inheritance well
```

**Problem 4: Deep inheritance hierarchies**
```cpp
Entity
  → Damageable
    → Moveable
      → Character
        → Player
        → Enemy
          → Zombie
            → FastZombie
            → SlowZombie
          → Skeleton
// 8 levels deep! Nightmare to maintain!
```

---

## The Solution: Component Pattern

**Build entities by composing components, not inheriting.**

Think of it like **LEGO blocks**:
- Each component is a block (health, transform, velocity)
- Snap blocks together to create entities
- Want different entity? Use different blocks
- No rigid hierarchy - pure flexibility

### Key Characteristics:
1. **Composition over inheritance** - entity HAS components, not IS-A type
2. **Reusable components** - same HealthComponent used by player, zombie, barrel
3. **Mix and match** - create any combination of components
4. **Data-driven** - define entities by their component list

---

## Implementation in Zombie Horde

### Component.h (Base Class)

```cpp
#ifndef COMPONENT_H
#define COMPONENT_H

class Entity;  // Forward declaration

class Component
{
protected:
    Entity* owner;  // Entity that owns this component
    bool enabled;
    
public:
    Component() : owner(nullptr), enabled(true) {}
    virtual ~Component() = default;
    
    // Lifecycle
    virtual void Init(Entity* ownerEntity) { owner = ownerEntity; }
    virtual void Update(float deltaTime) {}
    virtual void Draw() const {}
    
    // Enable/disable component
    void SetEnabled(bool value) { enabled = value; }
    bool IsEnabled() const { return enabled; }
    
    // Get owner
    Entity* GetOwner() const { return owner; }
};

#endif
```

---

### Entity.h (Component Container)

```cpp
#ifndef ENTITY_H
#define ENTITY_H

#include "Component.h"
#include <vector>
#include <memory>
#include <typeindex>
#include <unordered_map>

class Entity
{
private:
    std::vector<std::unique_ptr<Component>> components;
    std::unordered_map<std::type_index, Component*> componentMap;
    bool active;
    
public:
    Entity() : active(true) {}
    
    // Add a component
    template<typename T>
    T* AddComponent()
    {
        T* component = new T();
        component->Init(this);
        
        components.push_back(std::unique_ptr<Component>(component));
        componentMap[typeid(T)] = component;
        
        return component;
    }
    
    // Get a component (returns nullptr if not found)
    template<typename T>
    T* GetComponent()
    {
        auto it = componentMap.find(typeid(T));
        if (it != componentMap.end())
        {
            return static_cast<T*>(it->second);
        }
        return nullptr;
    }
    
    // Check if entity has component
    template<typename T>
    bool HasComponent()
    {
        return componentMap.find(typeid(T)) != componentMap.end();
    }
    
    // Remove a component
    template<typename T>
    void RemoveComponent()
    {
        auto it = componentMap.find(typeid(T));
        if (it != componentMap.end())
        {
            componentMap.erase(it);
            
            // Remove from vector (find by pointer)
            components.erase(
                std::remove_if(components.begin(), components.end(),
                    [&](const std::unique_ptr<Component>& c) {
                        return c.get() == it->second;
                    }),
                components.end()
            );
        }
    }
    
    // Update all components
    void Update(float deltaTime)
    {
        if (!active) return;
        
        for (auto& component : components)
        {
            if (component->IsEnabled())
            {
                component->Update(deltaTime);
            }
        }
    }
    
    // Draw all components
    void Draw() const
    {
        if (!active) return;
        
        for (auto& component : components)
        {
            if (component->IsEnabled())
            {
                component->Draw();
            }
        }
    }
    
    // Active state
    void SetActive(bool value) { active = value; }
    bool IsActive() const { return active; }
};

#endif
```

---

### TransformComponent.h

```cpp
#ifndef TRANSFORM_COMPONENT_H
#define TRANSFORM_COMPONENT_H

#include "Component.h"
#include "raylib.h"

class TransformComponent : public Component
{
private:
    Vector2 position;
    float rotation;  // In degrees
    float scale;
    
public:
    TransformComponent()
        : position({0, 0})
        , rotation(0.0f)
        , scale(1.0f)
    {}
    
    // Position
    void SetPosition(Vector2 pos) { position = pos; }
    void SetPosition(float x, float y) { position = {x, y}; }
    Vector2 GetPosition() const { return position; }
    
    // Rotation
    void SetRotation(float rot) { rotation = rot; }
    float GetRotation() const { return rotation; }
    
    // Scale
    void SetScale(float s) { scale = s; }
    float GetScale() const { return scale; }
    
    // Movement helpers
    void Translate(Vector2 offset)
    {
        position.x += offset.x;
        position.y += offset.y;
    }
    
    void Rotate(float degrees)
    {
        rotation += degrees;
    }
};

#endif
```

---

### VelocityComponent.h

```cpp
#ifndef VELOCITY_COMPONENT_H
#define VELOCITY_COMPONENT_H

#include "Component.h"
#include "TransformComponent.h"
#include "raylib.h"

class VelocityComponent : public Component
{
private:
    Vector2 velocity;
    float maxSpeed;
    float friction;  // 0 = no friction, 1 = instant stop
    
public:
    VelocityComponent()
        : velocity({0, 0})
        , maxSpeed(1000.0f)
        , friction(0.0f)
    {}
    
    void Update(float deltaTime) override
    {
        // Apply friction
        if (friction > 0)
        {
            velocity.x *= (1.0f - friction * deltaTime);
            velocity.y *= (1.0f - friction * deltaTime);
        }
        
        // Clamp to max speed
        float speed = sqrtf(velocity.x * velocity.x + velocity.y * velocity.y);
        if (speed > maxSpeed)
        {
            velocity.x = (velocity.x / speed) * maxSpeed;
            velocity.y = (velocity.y / speed) * maxSpeed;
        }
        
        // Move transform
        TransformComponent* transform = owner->GetComponent<TransformComponent>();
        if (transform)
        {
            transform->Translate(Vector2{
                velocity.x * deltaTime,
                velocity.y * deltaTime
            });
        }
    }
    
    // Velocity
    void SetVelocity(Vector2 vel) { velocity = vel; }
    void SetVelocity(float x, float y) { velocity = {x, y}; }
    Vector2 GetVelocity() const { return velocity; }
    
    void AddVelocity(Vector2 vel)
    {
        velocity.x += vel.x;
        velocity.y += vel.y;
    }
    
    // Settings
    void SetMaxSpeed(float speed) { maxSpeed = speed; }
    float GetMaxSpeed() const { return maxSpeed; }
    
    void SetFriction(float f) { friction = f; }
    float GetFriction() const { return friction; }
};

#endif
```

---

### HealthComponent.h

```cpp
#ifndef HEALTH_COMPONENT_H
#define HEALTH_COMPONENT_H

#include "Component.h"
#include "TransformComponent.h"
#include "raylib.h"

class HealthComponent : public Component
{
private:
    int health;
    int maxHealth;
    bool showHealthBar;
    float barWidth;
    float barHeight;
    
public:
    HealthComponent()
        : health(100)
        , maxHealth(100)
        , showHealthBar(true)
        , barWidth(40.0f)
        , barHeight(5.0f)
    {}
    
    void Draw() const override
    {
        if (!showHealthBar) return;
        
        TransformComponent* transform = owner->GetComponent<TransformComponent>();
        if (!transform) return;
        
        Vector2 pos = transform->GetPosition();
        
        // Calculate health bar position (above entity)
        float barX = pos.x - barWidth / 2;
        float barY = pos.y - 30;  // 30 pixels above
        
        // Background (red)
        DrawRectangle(barX, barY, barWidth, barHeight, RED);
        
        // Foreground (green, scaled by health)
        float healthPercent = (float)health / (float)maxHealth;
        DrawRectangle(barX, barY, barWidth * healthPercent, barHeight, GREEN);
        
        // Border
        DrawRectangleLines(barX, barY, barWidth, barHeight, BLACK);
    }
    
    // Health management
    void SetHealth(int hp)
    {
        health = hp;
        if (health > maxHealth) health = maxHealth;
        if (health < 0) health = 0;
    }
    
    void SetMaxHealth(int maxHp)
    {
        maxHealth = maxHp;
        if (health > maxHealth) health = maxHealth;
    }
    
    int GetHealth() const { return health; }
    int GetMaxHealth() const { return maxHealth; }
    
    void Heal(int amount)
    {
        health += amount;
        if (health > maxHealth) health = maxHealth;
    }
    
    void TakeDamage(int damage)
    {
        health -= damage;
        if (health < 0) health = 0;
    }
    
    bool IsDead() const { return health <= 0; }
    bool IsFullHealth() const { return health == maxHealth; }
    
    float GetHealthPercent() const
    {
        return (float)health / (float)maxHealth;
    }
    
    // Health bar display
    void SetShowHealthBar(bool show) { showHealthBar = show; }
    void SetHealthBarSize(float width, float height)
    {
        barWidth = width;
        barHeight = height;
    }
};

#endif
```

---

### RenderComponent.h

```cpp
#ifndef RENDER_COMPONENT_H
#define RENDER_COMPONENT_H

#include "Component.h"
#include "TransformComponent.h"
#include "raylib.h"

class RenderComponent : public Component
{
private:
    Color color;
    float radius;
    
public:
    RenderComponent()
        : color(WHITE)
        , radius(10.0f)
    {}
    
    void Draw() const override
    {
        TransformComponent* transform = owner->GetComponent<TransformComponent>();
        if (!transform) return;
        
        Vector2 pos = transform->GetPosition();
        float scale = transform->GetScale();
        
        DrawCircleV(pos, radius * scale, color);
    }
    
    void SetColor(Color col) { color = col; }
    Color GetColor() const { return color; }
    
    void SetRadius(float r) { radius = r; }
    float GetRadius() const { return radius; }
};

#endif
```

---

### ChaseComponent.h (AI Behavior)

```cpp
#ifndef CHASE_COMPONENT_H
#define CHASE_COMPONENT_H

#include "Component.h"
#include "TransformComponent.h"
#include "VelocityComponent.h"
#include "raylib.h"
#include <cmath>

class ChaseComponent : public Component
{
private:
    Vector2 targetPosition;
    float chaseSpeed;
    
public:
    ChaseComponent()
        : targetPosition({0, 0})
        , chaseSpeed(100.0f)
    {}
    
    void Update(float deltaTime) override
    {
        TransformComponent* transform = owner->GetComponent<TransformComponent>();
        VelocityComponent* velocity = owner->GetComponent<VelocityComponent>();
        
        if (!transform || !velocity) return;
        
        Vector2 currentPos = transform->GetPosition();
        
        // Calculate direction to target
        Vector2 direction = {
            targetPosition.x - currentPos.x,
            targetPosition.y - currentPos.y
        };
        
        // Normalize
        float length = sqrtf(direction.x * direction.x + direction.y * direction.y);
        if (length > 0)
        {
            direction.x /= length;
            direction.y /= length;
        }
        
        // Set velocity toward target
        velocity->SetVelocity(
            direction.x * chaseSpeed,
            direction.y * chaseSpeed
        );
    }
    
    void SetTarget(Vector2 target) { targetPosition = target; }
    Vector2 GetTarget() const { return targetPosition; }
    
    void SetChaseSpeed(float speed) { chaseSpeed = speed; }
    float GetChaseSpeed() const { return chaseSpeed; }
};

#endif
```

---

## How to Use It

### Creating a Zombie

```cpp
// Create entity
Entity* zombie = new Entity();

// Add components
TransformComponent* transform = zombie->AddComponent<TransformComponent>();
transform->SetPosition(100, 100);

VelocityComponent* velocity = zombie->AddComponent<VelocityComponent>();
velocity->SetMaxSpeed(50.0f);

HealthComponent* health = zombie->AddComponent<HealthComponent>();
health->SetMaxHealth(50);
health->SetHealth(50);

RenderComponent* render = zombie->AddComponent<RenderComponent>();
render->SetColor(GREEN);
render->SetRadius(20.0f);

ChaseComponent* chase = zombie->AddComponent<ChaseComponent>();
chase->SetChaseSpeed(50.0f);
chase->SetTarget(playerPosition);

// Update zombie every frame
zombie->Update(deltaTime);
zombie->Draw();
```

### Creating a Player

```cpp
Entity* player = new Entity();

// Player has similar components but different values
player->AddComponent<TransformComponent>()->SetPosition(400, 300);
player->AddComponent<VelocityComponent>()->SetMaxSpeed(200.0f);  // Faster than zombie
player->AddComponent<HealthComponent>()->SetMaxHealth(100);
player->AddComponent<RenderComponent>()->SetColor(BLUE);
// No ChaseComponent - player is controlled by input
```

### Creating a Barrel (Explosive)

```cpp
Entity* barrel = new Entity();

barrel->AddComponent<TransformComponent>()->SetPosition(200, 200);
// NO VelocityComponent - barrels don't move!
barrel->AddComponent<HealthComponent>()->SetMaxHealth(20);
barrel->AddComponent<RenderComponent>()->SetColor(ORANGE);
// Could add ExplosionComponent for when it dies
```

### Creating a Power-Up

```cpp
Entity* powerUp = new Entity();

powerUp->AddComponent<TransformComponent>()->SetPosition(300, 300);
powerUp->AddComponent<VelocityComponent>()->SetVelocity(0, 50);  // Floats down
// NO HealthComponent - power-ups can't be damaged!
powerUp->AddComponent<RenderComponent>()->SetColor(YELLOW);
```

---

## How It Works

### Before Components (Inheritance)

```
        Entity (position, velocity, health)
           │
    ┌──────┴──────┬──────────┬──────────┐
    │             │          │          │
  Player       Zombie     Barrel    PowerUp
  
Barrel forced to have velocity (doesn't move!)
PowerUp forced to have health (can't be damaged!)
Player and Zombie duplicate DrawHealthBar()
```

### After Components (Composition)

```
Components:
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Transform   │ │  Velocity   │ │   Health    │
└─────────────┘ └─────────────┘ └─────────────┘
┌─────────────┐ ┌─────────────┐
│   Render    │ │    Chase    │
└─────────────┘ └─────────────┘

Zombie = Transform + Velocity + Health + Render + Chase
Player = Transform + Velocity + Health + Render
Barrel = Transform + Health + Render
PowerUp = Transform + Velocity + Render

Each entity has exactly what it needs!
```

---

## Exercise 1: Basic Extension - Collision Component

**Goal:** Create a CollisionComponent for circle collision detection.

**Requirements:**
1. Create `CollisionComponent.h`
2. Store collision radius
3. Add method: `bool IsCollidingWith(Entity* other)`
4. Use formula: `distance < radius1 + radius2`
5. Test: Bullets hitting zombies

**Starter code:**
```cpp
class CollisionComponent : public Component
{
private:
    float radius;
    
public:
    CollisionComponent() : radius(10.0f) {}
    
    void SetRadius(float r) { radius = r; }
    float GetRadius() const { return radius; }
    
    bool IsCollidingWith(Entity* other)
    {
        // Get both transforms
        TransformComponent* myTransform = owner->GetComponent<TransformComponent>();
        TransformComponent* otherTransform = other->GetComponent<TransformComponent>();
        
        if (!myTransform || !otherTransform) return false;
        
        // Get other collision component
        CollisionComponent* otherCollision = other->GetComponent<CollisionComponent>();
        if (!otherCollision) return false;
        
        // Calculate distance
        Vector2 myPos = myTransform->GetPosition();
        Vector2 otherPos = otherTransform->GetPosition();
        
        float dx = myPos.x - otherPos.x;
        float dy = myPos.y - otherPos.y;
        float distance = sqrtf(dx * dx + dy * dy);
        
        // Check collision
        return distance < (radius + otherCollision->GetRadius());
    }
};
```

---

## Exercise 2: Intermediate Challenge - Lifetime Component

**Goal:** Create a component that automatically destroys entities after time.

**Requirements:**
1. Create `LifetimeComponent`
2. Countdown timer (e.g., 3 seconds)
3. When timer expires, deactivate entity
4. Use for: particles, temporary power-ups, bullet trails

**Starter code:**
```cpp
class LifetimeComponent : public Component
{
private:
    float lifetime;
    float elapsed;
    
public:
    LifetimeComponent() : lifetime(1.0f), elapsed(0.0f) {}
    
    void Update(float deltaTime) override
    {
        elapsed += deltaTime;
        
        if (elapsed >= lifetime)
        {
            owner->SetActive(false);  // Destroy entity
        }
    }
    
    void SetLifetime(float time) { lifetime = time; }
    void ResetTimer() { elapsed = 0.0f; }
};
```

**Test:** Bullet that disappears after 2 seconds.

---

## Exercise 3: Advanced - Animation Component

**Goal:** Create a component that animates sprite frames.

**Requirements:**
1. Store array of textures/frames
2. Frame rate (frames per second)
3. Loop or play once
4. Replace RenderComponent with sprite rendering

**Starter code:**
```cpp
class AnimationComponent : public Component
{
private:
    std::vector<Texture2D> frames;
    int currentFrame;
    float frameTime;
    float frameRate;
    float elapsed;
    bool loop;
    
public:
    AnimationComponent()
        : currentFrame(0)
        , frameTime(0.1f)
        , frameRate(10.0f)
        , elapsed(0.0f)
        , loop(true)
    {}
    
    void Update(float deltaTime) override
    {
        if (frames.empty()) return;
        
        elapsed += deltaTime;
        
        if (elapsed >= frameTime)
        {
            elapsed = 0.0f;
            currentFrame++;
            
            if (currentFrame >= frames.size())
            {
                if (loop)
                {
                    currentFrame = 0;
                }
                else
                {
                    currentFrame = frames.size() - 1;  // Stay on last frame
                }
            }
        }
    }
    
    void Draw() const override
    {
        if (frames.empty()) return;
        
        TransformComponent* transform = owner->GetComponent<TransformComponent>();
        if (!transform) return;
        
        Vector2 pos = transform->GetPosition();
        DrawTextureV(frames[currentFrame], pos, WHITE);
    }
    
    void AddFrame(Texture2D frame) { frames.push_back(frame); }
    void SetFrameRate(float fps) { frameRate = fps; frameTime = 1.0f / fps; }
    void SetLoop(bool looping) { loop = looping; }
};
```

---

## Exercise 4: Advanced (Optional) - Component Dependencies

**Goal:** Make components automatically add required dependencies.

**Problem:**
```cpp
// VelocityComponent needs TransformComponent
// But user forgot to add it!
Entity* entity = new Entity();
entity->AddComponent<VelocityComponent>();  // Won't work - no transform!
```

**Solution:**
Automatically add required components:

```cpp
class VelocityComponent : public Component
{
public:
    void Init(Entity* ownerEntity) override
    {
        Component::Init(ownerEntity);
        
        // Ensure TransformComponent exists
        if (!owner->HasComponent<TransformComponent>())
        {
            owner->AddComponent<TransformComponent>();
        }
    }
};
```

**Challenge:** Implement this for all components that have dependencies. What if there's a circular dependency?

---

## Common Mistakes

### ❌ Mistake 1: Forgetting to check for nullptr

```cpp
// WRONG - component might not exist!
void Update()
{
    TransformComponent* transform = owner->GetComponent<TransformComponent>();
    transform->SetPosition(100, 100);  // CRASH if nullptr!
}
```

**Fix:** Always check
```cpp
// CORRECT
void Update()
{
    TransformComponent* transform = owner->GetComponent<TransformComponent>();
    if (transform)  // Check first!
    {
        transform->SetPosition(100, 100);
    }
}
```

---

### ❌ Mistake 2: Tight coupling between components

```cpp
// WRONG - HealthComponent directly accessing VelocityComponent
class HealthComponent : public Component
{
    void TakeDamage(int damage)
    {
        health -= damage;
        
        // BAD: Reaching into another component
        VelocityComponent* vel = owner->GetComponent<VelocityComponent>();
        vel->SetVelocity(100, 0);  // Knockback
    }
};
```

**Fix:** Use events or messages
```cpp
// CORRECT - fire event instead
void TakeDamage(int damage)
{
    health -= damage;
    
    // Let other components react to damage event
    GameEvents::Instance().OnPlayerHit.Invoke({damage, position});
}

// KnockbackComponent listens to hit events
class KnockbackComponent : public Component
{
    void Init(Entity* entity) override
    {
        GameEvents::Instance().OnPlayerHit.Subscribe([this](auto& data) {
            // Apply knockback velocity
        });
    }
};
```

---

### ❌ Mistake 3: God components

```cpp
// WRONG - one component doing everything
class GameplayComponent : public Component
{
    void Update() override
    {
        // Handle movement
        // Handle shooting
        // Handle health
        // Handle collision
        // Handle AI
        // 500 lines of code...
    }
};
```

**Fix:** Split into focused components
```cpp
// CORRECT - each component has one job
entity->AddComponent<MovementComponent>();
entity->AddComponent<ShootingComponent>();
entity->AddComponent<HealthComponent>();
entity->AddComponent<CollisionComponent>();
entity->AddComponent<AIComponent>();
```

---

### ❌ Mistake 4: Not using Entity as the owner

```cpp
// WRONG - directly storing other components
class VelocityComponent : public Component
{
    TransformComponent* transform;  // Don't store direct pointer!
    
    void Init(Entity* entity)
    {
        transform = entity->GetComponent<TransformComponent>();
        // What if transform is added AFTER this component?
        // What if transform is removed?
    }
};
```

**Fix:** Look up components each time
```cpp
// CORRECT - look up when needed
class VelocityComponent : public Component
{
    void Update(float deltaTime) override
    {
        // Look up every time (it's fast with the map)
        TransformComponent* transform = owner->GetComponent<TransformComponent>();
        if (transform)
        {
            // Use it
        }
    }
};
```

---

### ❌ Mistake 5: Memory management confusion

```cpp
// WRONG - who owns this component?
Component* comp = new HealthComponent();
entity->AddComponent(comp);  // Entity now owns it
delete comp;  // CRASH! Entity already owns it!
```

**Fix:** Let Entity manage memory
```cpp
// CORRECT - Entity creates and owns components
HealthComponent* health = entity->AddComponent<HealthComponent>();
// Entity will delete it when destroyed
```

---

## When to Use Component Pattern

### ✅ Good Use Cases:
- **Game entities** (player, enemies, props, power-ups)
- **Flexible behavior** (many possible combinations)
- **Data-driven design** (define entities in files)
- **Plugin systems** (add components at runtime)
- **Modular code** (easy to test individual components)

### ❌ Bad Use Cases:
- **Simple objects** with no variation (if all entities are identical, just use a class)
- **Performance-critical code** (component lookup has overhead)
- **Tightly coupled systems** (if components always go together, just make one class)

### The Rule of Thumb:
**"If you find yourself writing 'HasA HasA HasA' instead of 'IsA IsA IsA', use components."**

---

## Component Pattern Variations

### 1. Simple Components (What we built)
```cpp
Entity has Components
Components update themselves
```
- Simple to understand
- Good for small-medium games

### 2. Entity Component System (ECS)
```cpp
Entities are just IDs
Components are pure data
Systems process components
```
- Max performance
- Used in AAA engines (Unity DOTS, Unreal Mass)
- Complex to implement

### 3. Component Messages
```cpp
Components communicate via messages
entity->SendMessage("TakeDamage", 10);
```
- More decoupled
- Slower than direct calls

---

## Summary

✅ **Component Pattern** builds entities through composition<br>
✅ **Reusable components** - write once, use everywhere<br>
✅ **Flexible** - any combination of components<br>
✅ **Modular** - easy to add/remove/test components<br>
✅ **Data-driven** - define entities by component list

**The pattern you just learned:**
```cpp
// 1. Create entity
Entity* entity = new Entity();

// 2. Add components
entity->AddComponent<TransformComponent>();
entity->AddComponent<VelocityComponent>();
entity->AddComponent<HealthComponent>();

// 3. Update entity (updates all components)
entity->Update(deltaTime);
entity->Draw();
```

**Before Components:**
```cpp
// Rigid hierarchy
class Zombie : public Enemy : public Character : public Entity
{
    // 7 levels of inheritance
    // Can't reuse health bar code
    // Can't make exploding zombie without new class
};
```

**After Components:**
```cpp
// Flexible composition
Entity* zombie = new Entity();
zombie->AddComponent<TransformComponent>();
zombie->AddComponent<VelocityComponent>();
zombie->AddComponent<HealthComponent>();  // ← Reusable!
zombie->AddComponent<RenderComponent>();
zombie->AddComponent<ChaseComponent>();

// Want exploding zombie? Just add explosion component!
zombie->AddComponent<ExplosionComponent>();
```

**Next Pattern:** Strategy Pattern - swappable AI behaviors (ChaseStrategy, FleeStrategy, CircleStrategy)!