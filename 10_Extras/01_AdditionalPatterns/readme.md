# Additional Design Patterns

These patterns didn't make it into the main Zombie Horde course, but they're valuable to know about! Here's a quick overview of each with examples of when you might use them.

---

## 1. Flyweight Pattern

**What it is:** Share common data between many objects to save memory.

**The Problem:**
```cpp
// 1000 zombies, each stores a 2MB texture
class Zombie
{
    Texture2D texture;  // 2MB × 1000 = 2000MB (2GB!)
    Vector2 position;   // Unique per zombie
};
```

**The Solution:**
```cpp
// Shared data (intrinsic state)
class ZombieType
{
    Texture2D texture;      // Shared by all slow zombies
    float baseSpeed;
    int baseHealth;
};

// Unique data (extrinsic state)
class Zombie
{
    ZombieType* type;       // Pointer to shared data (4 bytes)
    Vector2 position;       // Unique per zombie
    int currentHealth;
};

// Now: 2MB + (1000 × 20 bytes) = 2.02MB instead of 2GB!
```

**When to use:**
- Thousands of similar objects (particles, tiles, grass, trees)
- Most data is shared (texture, sounds, stats)
- Memory is limited

**Real examples:**
- Tile-based games: 10,000 tiles share 10 textures
- Particle systems: 5,000 particles share 1 texture
- Forest: 1,000 trees share 5 tree models

---

## 2. Decorator Pattern

**What it is:** Add functionality to objects dynamically without changing their class.

**The Problem:**
```cpp
// Want zombie with armor? New class.
// Want zombie with fire damage? New class.
// Want zombie with armor AND fire? Another new class!
class ArmoredZombie : public Zombie { };
class FireZombie : public Zombie { };
class ArmoredFireZombie : public Zombie { };  // Combinatorial explosion!
```

**The Solution:**
```cpp
class Zombie
{
    virtual int GetDamage() { return baseDamage; }
};

// Decorator adds armor
class ArmorDecorator : public Zombie
{
    Zombie* wrappedZombie;
    
    int GetDamage() override
    {
        return wrappedZombie->GetDamage() + 10;  // +10 damage from armor
    }
};

// Decorator adds fire
class FireDecorator : public Zombie
{
    Zombie* wrappedZombie;
    
    int GetDamage() override
    {
        return wrappedZombie->GetDamage() * 1.5;  // +50% fire damage
    }
};

// Combine them!
Zombie* zombie = new Zombie();
zombie = new ArmorDecorator(zombie);      // Add armor
zombie = new FireDecorator(zombie);       // Add fire
// Result: (baseDamage + 10) * 1.5
```

**When to use:**
- Adding abilities/power-ups to player
- Buff/debuff systems
- Equipment systems (sword + fire enchantment + poison enchantment)

**Real examples:**
- RPG equipment: Sword → +Fire → +Poison → +Critical
- Power-ups: Player → SpeedBoost → InvincibilityShield → DoubleJump
- Java I/O streams: `BufferedReader(InputStreamReader(FileInputStream()))`

---

## 3. Adapter Pattern

**What it is:** Make incompatible interfaces work together.

**The Problem:**
```cpp
// Your audio system expects WAV files
class AudioSystem
{
    void PlayWAV(const char* filename);
};

// But you have MP3 files!
class MP3File
{
    void DecodeMP3();
    byte* GetPCMData();
};
```

**The Solution:**
```cpp
// Adapter converts MP3 to WAV format
class MP3Adapter
{
    MP3File mp3;
    
    void PlayWAV(const char* filename)
    {
        mp3.Load(filename);
        mp3.DecodeMP3();
        byte* pcmData = mp3.GetPCMData();
        audioSystem.PlayRawPCM(pcmData);  // Convert and play
    }
};

// Now MP3 files work with your system!
AudioSystem audio;
audio.Play(new MP3Adapter("music.mp3"));
```

**When to use:**
- Using third-party libraries with different APIs
- Legacy code integration
- Platform-specific code (Windows ↔ Linux)

**Real examples:**
- Physics engines: Box2D → Your game's coordinate system
- Input: Gamepad API → Your InputHandler
- Graphics: OpenGL → DirectX wrapper

---

## 4. Prototype Pattern

**What it is:** Clone objects instead of creating from scratch.

**The Problem:**
```cpp
// Creating complex enemy takes lots of setup
Enemy* CreateBoss()
{
    Enemy* boss = new Enemy();
    boss->SetHealth(1000);
    boss->SetSpeed(50);
    boss->SetWeapon(new Minigun());
    boss->SetArmor(new HeavyArmor());
    boss->SetAI(new AggressiveAI());
    boss->AddAbility(new Charge());
    boss->AddAbility(new RageMode());
    // 20 more lines...
}
```

**The Solution:**
```cpp
class Enemy
{
    virtual Enemy* Clone()
    {
        Enemy* copy = new Enemy();
        copy->health = this->health;
        copy->speed = this->speed;
        copy->weapon = this->weapon->Clone();
        // Copy everything
        return copy;
    }
};

// Create boss template once
Enemy* bossTemplate = CreateBossTemplate();

// Clone it many times
Enemy* boss1 = bossTemplate->Clone();  // Fast!
Enemy* boss2 = bossTemplate->Clone();  // Fast!
Enemy* boss3 = bossTemplate->Clone();  // Fast!
```

**When to use:**
- Creating many similar objects
- Object creation is expensive
- Templates/prefabs system

**Real examples:**
- Unity Prefabs: Create template, instantiate many copies
- Particle systems: Clone particle template 1000x
- Level editor: Clone enemy templates

---

## 5. Builder Pattern

**What it is:** Construct complex objects step-by-step.

**The Problem:**
```cpp
// Too many constructor parameters!
Character* hero = new Character(
    "Hero",           // name
    100,              // health
    50,               // mana
    10,               // strength
    5,                // dexterity
    8,                // intelligence
    new Sword(),      // weapon
    new LeatherArmor(),  // armor
    new FireSpell(),  // spell1
    new HealSpell(),  // spell2
    {10, 20},         // position
    TEAM_GOOD         // team
);
// Hard to read! What's 50? What's 8?
```

**The Solution:**
```cpp
class CharacterBuilder
{
    Character* character;
    
public:
    CharacterBuilder() { character = new Character(); }
    
    CharacterBuilder* SetName(const char* name)
    {
        character->name = name;
        return this;
    }
    
    CharacterBuilder* SetHealth(int hp)
    {
        character->health = hp;
        return this;
    }
    
    CharacterBuilder* SetWeapon(Weapon* weapon)
    {
        character->weapon = weapon;
        return this;
    }
    
    Character* Build() { return character; }
};

// Readable and flexible!
Character* hero = CharacterBuilder()
    .SetName("Hero")
    .SetHealth(100)
    .SetMana(50)
    .SetWeapon(new Sword())
    .SetArmor(new LeatherArmor())
    .Build();
```

**When to use:**
- Many optional parameters
- Complex object construction
- Want readable, fluent API

**Real examples:**
- Query builders: `Query().Where("health > 50").OrderBy("name").Limit(10)`
- Level builders: `Level().AddRoom().AddEnemies(5).AddTreasure().Build()`
- UI builders: `Button().SetText("Click").SetColor(RED).SetSize(100, 50)`

---

## 6. Proxy Pattern

**What it is:** Control access to an object through a surrogate.

**The Problem:**
```cpp
// Loading large texture takes 2 seconds
class Texture
{
    byte* pixelData;  // 50MB
    
    Texture(const char* file)
    {
        pixelData = LoadFromDisk(file);  // SLOW!
    }
};

// Game freezes for 2 seconds while loading!
Texture* background = new Texture("background.png");
```

**The Solution:**
```cpp
class TextureProxy
{
    Texture* realTexture;
    const char* filename;
    bool loaded;
    
public:
    TextureProxy(const char* file)
        : filename(file), realTexture(nullptr), loaded(false)
    {}
    
    void Draw()
    {
        if (!loaded)
        {
            realTexture = new Texture(filename);  // Load on first use
            loaded = true;
        }
        
        realTexture->Draw();
    }
};

// Instant creation!
TextureProxy* background = new TextureProxy("background.png");
// Texture only loads when first drawn (lazy loading)
```

**When to use:**
- Lazy loading (load on demand)
- Access control (check permissions)
- Remote objects (network proxy)
- Caching

**Real examples:**
- Virtual proxy: Load textures/models only when visible
- Protection proxy: Check if player can open door
- Remote proxy: Network player looks like local player
- Smart pointer: `std::shared_ptr` is a proxy

---

## 7. Template Method Pattern

**What it is:** Define algorithm skeleton, let subclasses fill in steps.

**The Problem:**
```cpp
class Enemy
{
    void Update()
    {
        Move();
        Attack();
        CheckDeath();
    }
};

// Zombie and Skeleton have same structure but different implementations
```

**The Solution:**
```cpp
class Enemy
{
public:
    // Template method (algorithm skeleton)
    void Update()
    {
        Move();           // Step 1
        Attack();         // Step 2
        CheckDeath();     // Step 3
        PlaySound();      // Step 4
    }
    
protected:
    // Steps implemented by subclasses
    virtual void Move() = 0;
    virtual void Attack() = 0;
    virtual void PlaySound() = 0;
    
    // Common implementation
    void CheckDeath()
    {
        if (health <= 0) isAlive = false;
    }
};

class Zombie : public Enemy
{
    void Move() override { /* Shuffle toward player */ }
    void Attack() override { /* Bite */ }
    void PlaySound() override { /* Groan */ }
};

class Skeleton : public Enemy
{
    void Move() override { /* Run toward player */ }
    void Attack() override { /* Shoot arrow */ }
    void PlaySound() override { /* Rattle bones */ }
};
```

**When to use:**
- Multiple classes with same algorithm structure
- Want to enforce certain steps
- Some steps are common, some vary

**Real examples:**
- Game loop: `Initialize()` → `Update()` → `Draw()` → `Cleanup()`
- File readers: `Open()` → `ReadHeader()` → `ReadData()` → `Close()`
- AI: `Sense()` → `Think()` → `Act()`

---

## 8. Visitor Pattern

**What it is:** Add operations to classes without modifying them.

**The Problem:**
```cpp
// Want to add SaveToFile() to all entities?
// Modify every class!
class Player
{
    void SaveToFile() { /* ... */ }  // Add to Player
};

class Enemy
{
    void SaveToFile() { /* ... */ }  // Add to Enemy
};

// Want to add DrawDebugInfo()? Modify all classes AGAIN!
```

**The Solution:**
```cpp
class Visitor
{
    virtual void Visit(Player* player) = 0;
    virtual void Visit(Enemy* enemy) = 0;
};

class SaveVisitor : public Visitor
{
    void Visit(Player* player) override
    {
        // Save player to file
    }
    
    void Visit(Enemy* enemy) override
    {
        // Save enemy to file
    }
};

class DebugDrawVisitor : public Visitor
{
    void Visit(Player* player) override
    {
        // Draw player debug info
    }
    
    void Visit(Enemy* enemy) override
    {
        // Draw enemy debug info
    }
};

// Add operations without modifying classes!
SaveVisitor saver;
player->Accept(&saver);
enemy->Accept(&saver);
```

**When to use:**
- Many operations on stable class hierarchy
- Operations don't belong in the classes
- Want to add operations without modifying classes

**Real examples:**
- Compilers: AST visitor (parse, optimize, generate code)
- Scene graphs: Render visitor, collision visitor, save visitor
- UI: Layout visitor, draw visitor, event visitor

---

## 9. Memento Pattern

**What it is:** Save and restore object state without exposing internals.

**The Problem:**
```cpp
// Save game system
class Game
{
private:
    int level;
    int score;
    Vector2 playerPos;
    std::vector<Enemy> enemies;
    
public:
    // How to save without exposing private data?
};
```

**The Solution:**
```cpp
class GameMemento
{
    friend class Game;
private:
    int level;
    int score;
    Vector2 playerPos;
    std::vector<Enemy> enemies;
};

class Game
{
public:
    GameMemento* Save()
    {
        GameMemento* memento = new GameMemento();
        memento->level = level;
        memento->score = score;
        memento->playerPos = playerPos;
        memento->enemies = enemies;
        return memento;
    }
    
    void Restore(GameMemento* memento)
    {
        level = memento->level;
        score = memento->score;
        playerPos = memento->playerPos;
        enemies = memento->enemies;
    }
};

// Use it
GameMemento* save = game.Save();      // Save state
// ... play for a while ...
game.Restore(save);                   // Load state
```

**When to use:**
- Save/load system
- Undo/redo (store mementos)
- Checkpoints
- Snapshots for replay

**Real examples:**
- Save games: Store entire game state
- Undo in editor: Store state before each action
- Time rewind: Store memento every frame

---

## 10. Chain of Responsibility Pattern

**What it is:** Pass request along chain until someone handles it.

**The Problem:**
```cpp
// Who handles this input?
void HandleInput(int key)
{
    if (pauseMenu->IsOpen())
    {
        pauseMenu->HandleInput(key);
    }
    else if (inventory->IsOpen())
    {
        inventory->HandleInput(key);
    }
    else if (dialogue->IsActive())
    {
        dialogue->HandleInput(key);
    }
    else
    {
        player->HandleInput(key);
    }
}
```

**The Solution:**
```cpp
class InputHandler
{
protected:
    InputHandler* next;
    
public:
    void SetNext(InputHandler* nextHandler) { next = nextHandler; }
    
    virtual bool HandleInput(int key)
    {
        if (next)
        {
            return next->HandleInput(key);  // Pass to next
        }
        return false;
    }
};

class PauseMenuHandler : public InputHandler
{
    bool HandleInput(int key) override
    {
        if (isOpen)
        {
            // Handle input
            return true;  // Handled!
        }
        return InputHandler::HandleInput(key);  // Pass along
    }
};

// Set up chain
pauseMenu->SetNext(inventory);
inventory->SetNext(dialogue);
dialogue->SetNext(player);

// Input automatically goes to correct handler
pauseMenu->HandleInput(KEY_ESCAPE);
```

**When to use:**
- Event handling with priority
- Multiple potential handlers
- Don't know who will handle it

**Real examples:**
- UI input: Top window → parent window → root
- Collision: Check player → enemy → world
- Exception handling in some languages

---

## Pattern Selection Guide

| Need to... | Use Pattern |
|------------|-------------|
| Share data between thousands of objects | **Flyweight** |
| Add abilities dynamically | **Decorator** |
| Make incompatible interfaces work | **Adapter** |
| Clone complex objects | **Prototype** |
| Build objects with many parameters | **Builder** |
| Control access / lazy load | **Proxy** |
| Define algorithm skeleton | **Template Method** |
| Add operations without modifying classes | **Visitor** |
| Save and restore state | **Memento** |
| Pass request along chain | **Chain of Responsibility** |

---

## Remember

**Not every problem needs a pattern!**

✅ Use patterns when they solve a real problem<br>
❌ Don't force patterns just to use them<br>
✅ Start simple, refactor to patterns when needed<br>
❌ Don't over-engineer

**"The best code is code that's easy to understand and maintain."**