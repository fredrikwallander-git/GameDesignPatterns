# Bonus Content & Fun Extras 🎮

## 🎨 Visual Effects & Polish

### 1. Screen Shake
**Make impacts feel powerful!**

```cpp
class ScreenShake
{
private:
    float intensity;
    float duration;
    float elapsed;
    
public:
    void Shake(float power, float time)
    {
        intensity = power;
        duration = time;
        elapsed = 0;
    }
    
    void Update(float deltaTime)
    {
        if (elapsed < duration)
        {
            elapsed += deltaTime;
        }
    }
    
    Vector2 GetOffset()
    {
        if (elapsed >= duration) return {0, 0};
        
        float progress = elapsed / duration;
        float currentIntensity = intensity * (1.0f - progress);  // Fade out
        
        return {
            GetRandomValue(-100, 100) / 100.0f * currentIntensity * 10.0f,
            GetRandomValue(-100, 100) / 100.0f * currentIntensity * 10.0f
        };
    }
};

// Use in camera
BeginMode2D(camera);
Vector2 shake = screenShake.GetOffset();
camera.target = Vector2Add(player.position, shake);
```

**When to use:** Explosions, damage, boss attacks, impacts

---

### 2. Particle System
**Blood splatter, explosions, sparkles!**

```cpp
struct Particle
{
    Vector2 position;
    Vector2 velocity;
    Color color;
    float lifetime;
    float age;
    float size;
    bool active;
};

void SpawnExplosion(Vector2 position)
{
    for (int i = 0; i < 50; i++)
    {
        float angle = (i / 50.0f) * 2 * PI;
        Particle p;
        p.position = position;
        p.velocity = {cos(angle) * 200.0f, sin(angle) * 200.0f};
        p.color = (i % 2 == 0) ? ORANGE : RED;
        p.lifetime = 1.0f;
        p.size = GetRandomValue(5, 15);
        particlePool.Add(p);
    }
}
```

**Effects to try:**
- Blood splatter when zombie dies
- Muzzle flash when shooting
- Dust when walking
- Sparkles when collecting items

---

### 3. Chromatic Aberration
**Damage flash effect!**

```cpp
void DrawChromaticAberration()
{
    // Draw red channel shifted left
    DrawTextureV(gameTexture, {-2, 0}, RED);
    // Draw green channel normal
    DrawTextureV(gameTexture, {0, 0}, GREEN);
    // Draw blue channel shifted right
    DrawTextureV(gameTexture, {2, 0}, BLUE);
}

// Use when player takes damage
if (justTookDamage)
{
    DrawChromaticAberration();
}
```

---

### 4. Slow Motion
**Dramatic moments!**

```cpp
class TimeManager
{
    float timeScale = 1.0f;
    
public:
    void SetTimeScale(float scale) { timeScale = scale; }
    
    float GetDeltaTime()
    {
        return GetFrameTime() * timeScale;
    }
};

// Use it
if (nearDeath)
{
    timeManager.SetTimeScale(0.3f);  // 30% speed (slow-mo)
}
```

**When to use:** Low health, special moves, bullet time

---

### 5. Floating Damage Numbers
**Show damage visually!**

```cpp
struct FloatingText
{
    const char* text;
    Vector2 position;
    Color color;
    float lifetime;
    float velocity;
};

void ShowDamage(int damage, Vector2 position)
{
    FloatingText text;
    text.text = TextFormat("-%d", damage);
    text.position = position;
    text.color = RED;
    text.lifetime = 1.0f;
    text.velocity = -50.0f;  // Float upward
    
    floatingTexts.push_back(text);
}

void UpdateFloatingTexts(float deltaTime)
{
    for (auto& text : floatingTexts)
    {
        text.position.y += text.velocity * deltaTime;
        text.lifetime -= deltaTime;
        
        // Fade out
        float alpha = text.lifetime;
        text.color.a = (unsigned char)(alpha * 255);
    }
}
```

---

## 🎵 Audio Enhancements

### 6. Dynamic Music Layers
**Music intensity based on action!**

```cpp
class MusicManager
{
    Music baseLayer;
    Music actionLayer;
    Music bossLayer;
    
public:
    void Update()
    {
        int enemyCount = GetEnemyCount();
        bool bossActive = IsBossActive();
        
        // Adjust volumes based on action
        SetMusicVolume(baseLayer, 1.0f);
        SetMusicVolume(actionLayer, enemyCount > 5 ? 0.7f : 0.0f);
        SetMusicVolume(bossLayer, bossActive ? 1.0f : 0.0f);
    }
};
```

---

### 7. 3D Positional Audio
**Sound comes from where zombies are!**

```cpp
void UpdateAudio()
{
    Vector2 listenerPos = player.position;
    
    for (Zombie& zombie : zombies)
    {
        float distance = Vector2Distance(listenerPos, zombie.position);
        float volume = 1.0f / (distance / 100.0f);  // Inverse distance
        
        // Pan based on position
        Vector2 direction = zombie.position - listenerPos;
        float pan = direction.x / 400.0f;  // -1 (left) to +1 (right)
        
        SetSoundVolume(zombie.groanSound, volume);
        SetSoundPan(zombie.groanSound, pan);
    }
}
```

---

## 🎯 Gameplay Enhancements

### 8. Combo System
**Reward rapid kills!**

```cpp
class ComboSystem
{
    int comboCount = 0;
    float comboTimer = 0.0f;
    float comboWindow = 2.0f;
    
public:
    void OnKill()
    {
        comboCount++;
        comboTimer = 0.0f;
        
        if (comboCount >= 3)
        {
            int bonus = comboCount * 10;
            ShowFloatingText(TextFormat("COMBO x%d! +%d", comboCount, bonus));
        }
    }
    
    void Update(float deltaTime)
    {
        if (comboCount > 0)
        {
            comboTimer += deltaTime;
            if (comboTimer > comboWindow)
            {
                comboCount = 0;  // Reset
            }
        }
    }
};
```

---

### 9. Achievements System
**Track player milestones!**

```cpp
class Achievement
{
public:
    std::string id;
    std::string name;
    std::string description;
    bool unlocked;
    
    std::function<bool()> checkCondition;
};

// Define achievements
Achievement first100 = {
    "CENTURION",
    "Centurion",
    "Kill 100 zombies",
    false,
    []() { return GetTotalKills() >= 100; }
};

Achievement survivalist = {
    "SURVIVALIST",
    "Survivalist",
    "Survive 10 waves",
    false,
    []() { return GetCurrentWave() >= 10; }
};

void CheckAchievements()
{
    for (Achievement& ach : achievements)
    {
        if (!ach.unlocked && ach.checkCondition())
        {
            UnlockAchievement(ach);
        }
    }
}
```

---

### 10. Power-Up System
**Temporary boosts!**

```cpp
enum PowerUpType { SPEED, INVINCIBILITY, RAPID_FIRE, MAGNET };

class PowerUp
{
    PowerUpType type;
    float duration;
    float remaining;
    
public:
    void Activate(Player* player)
    {
        switch (type)
        {
            case SPEED:
                player->speed *= 2.0f;
                break;
            case INVINCIBILITY:
                player->invincible = true;
                break;
            case RAPID_FIRE:
                player->fireRate *= 3.0f;
                break;
        }
    }
    
    void Deactivate(Player* player)
    {
        // Restore original values
    }
};
```

---

## 🎮 Gameplay Modes

### 11. Wave-Based Difficulty Scaling
**Gets harder over time!**

```cpp
int GetZombieCount(int wave)
{
    return 5 + (wave * 3);  // 5, 8, 11, 14...
}

float GetZombieHealth(int wave)
{
    return 50 * (1.0f + wave * 0.2f);  // 50, 60, 70, 80...
}

float GetZombieSpeed(int wave)
{
    return 80 * (1.0f + wave * 0.1f);  // 80, 88, 96...
}

void SpawnWave(int wave)
{
    int count = GetZombieCount(wave);
    
    for (int i = 0; i < count; i++)
    {
        Zombie* zombie = zombiePool.Spawn();
        zombie->health = GetZombieHealth(wave);
        zombie->speed = GetZombieSpeed(wave);
    }
}
```

---

### 12. Boss Waves
**Every 5 waves, spawn boss!**

```cpp
void SpawnWave(int wave)
{
    if (wave % 5 == 0)
    {
        SpawnBoss();
        PlayMusic(bossMusic);
    }
    else
    {
        SpawnNormalWave(wave);
    }
}

void SpawnBoss()
{
    Zombie* boss = zombiePool.Spawn();
    boss->type = BOSS;
    boss->health = 1000;
    boss->size = 50.0f;
    boss->color = PURPLE;
    boss->speed = 40.0f;
    boss->damage = 50;
}
```

---

## 🎨 Visual Themes

### 13. Night/Day Cycle
**Changes atmosphere!**

```cpp
class DayNightCycle
{
    float timeOfDay = 0.0f;  // 0 = midnight, 0.5 = noon, 1.0 = midnight
    float cycleSpeed = 0.01f;
    
public:
    void Update(float deltaTime)
    {
        timeOfDay += cycleSpeed * deltaTime;
        if (timeOfDay > 1.0f) timeOfDay = 0.0f;
    }
    
    Color GetSkyColor()
    {
        if (timeOfDay < 0.25f)  // Night
            return {20, 20, 40, 255};
        else if (timeOfDay < 0.5f)  // Dawn
            return {100, 50, 80, 255};
        else if (timeOfDay < 0.75f)  // Day
            return {135, 206, 235, 255};
        else  // Dusk
            return {255, 140, 0, 255};
    }
};
```

---

### 14. Post-Processing Effects
**Shader-like effects!**

```cpp
// Vignette effect
void DrawVignette()
{
    for (int i = 0; i < 100; i++)
    {
        float radius = 800 - (i * 8);
        float alpha = i / 100.0f;
        DrawRing({400, 300}, radius, radius + 8, 0, 360, 32, 
                 Fade(BLACK, alpha));
    }
}

// Scanlines (retro effect)
void DrawScanlines()
{
    for (int y = 0; y < 600; y += 4)
    {
        DrawRectangle(0, y, 800, 2, Fade(BLACK, 0.3f));
    }
}
```

---

## 📊 Analytics & Debug

### 15. Performance Monitor
**Track FPS and memory!**

```cpp
class PerformanceMonitor
{
    int frameCount = 0;
    float elapsed = 0.0f;
    int avgFPS = 60;
    
public:
    void Update(float deltaTime)
    {
        frameCount++;
        elapsed += deltaTime;
        
        if (elapsed >= 1.0f)
        {
            avgFPS = frameCount;
            frameCount = 0;
            elapsed = 0.0f;
        }
    }
    
    void Draw()
    {
        DrawText(TextFormat("FPS: %d", avgFPS), 10, 10, 20, 
                 avgFPS < 30 ? RED : GREEN);
        DrawText(TextFormat("Entities: %d", GetEntityCount()), 10, 35, 20, WHITE);
        DrawText(TextFormat("Draw Calls: %d", GetDrawCallCount()), 10, 60, 20, WHITE);
    }
};
```

---

### 16. Heatmap Visualization
**Where do players die most?**

```cpp
std::vector<Vector2> deathPositions;

void OnPlayerDeath(Vector2 position)
{
    deathPositions.push_back(position);
}

void DrawHeatmap()
{
    for (Vector2 pos : deathPositions)
    {
        DrawCircle(pos.x, pos.y, 30, Fade(RED, 0.3f));
    }
}
```

---

## 🎓 Learning Tools

### 17. Tutorial System
**Teach players as they play!**

```cpp
class Tutorial
{
    int currentStep = 0;
    
    struct Step
    {
        const char* text;
        std::function<bool()> completed;
    };
    
    std::vector<Step> steps = {
        {"Move with WASD", []() { return playerMoved; }},
        {"Shoot with SPACE", []() { return playerShot; }},
        {"Kill a zombie", []() { return zombieKilled; }},
        {"Survive wave 1", []() { return wave >= 2; }}
    };
    
public:
    void Update()
    {
        if (currentStep < steps.size())
        {
            if (steps[currentStep].completed())
            {
                currentStep++;
            }
        }
    }
    
    void Draw()
    {
        if (currentStep < steps.size())
        {
            DrawRectangle(200, 500, 400, 60, Fade(BLACK, 0.7f));
            DrawText(steps[currentStep].text, 220, 515, 20, WHITE);
        }
    }
};
```

---

### 18. Replay System
**Watch your best runs!**

Already covered in Command Pattern, but add:
- Save top 3 replays
- Replay viewer with speed control
- Ghost player during replay

---

## 🌟 Fun Extras

### 19. Easter Eggs
**Hidden secrets!**

```cpp
// Super Code: ↑ ↑ ↓ ↓ ← → ← → B A
int superIndex = 0;
int superCode[] = {KEY_UP, KEY_UP, KEY_DOWN, KEY_DOWN, 
                    KEY_LEFT, KEY_RIGHT, KEY_LEFT, KEY_RIGHT, 
                    KEY_B, KEY_A};

void CheckSuperCode()
{
    int key = GetKeyPressed();
    
    if (key == superCode[superIndex])
    {
        superIndex++;
        
        if (superIndex == 10)
        {
            ActivateGodMode();
            superIndex = 0;
        }
    }
    else if (key != 0)
    {
        superIndex = 0;
    }
}
```

---

### 20. Randomized Elements
**Every playthrough is different!**

```cpp
// Random zombie skins
Color GetRandomZombieColor()
{
    Color colors[] = {GREEN, DARKGREEN, LIME, OLIVE};
    return colors[GetRandomValue(0, 3)];
}

// Random level layouts
void GenerateRandomLevel()
{
    for (int i = 0; i < GetRandomValue(3, 8); i++)
    {
        Vector2 obstaclePos = {
            GetRandomValue(0, 800),
            GetRandomValue(0, 600)
        };
        SpawnObstacle(obstaclePos);
    }
}

// Random events
void TriggerRandomEvent()
{
    int event = GetRandomValue(0, 4);
    
    switch (event)
    {
        case 0: SpawnSupplyDrop(); break;
        case 1: SpawnExtraEnemies(); break;
        case 2: GivePlayerPowerUp(); break;
        case 3: StartFogOfWar(); break;
    }
}
```

---

## 🏆 Challenge Ideas

### 21. Optional Challenges
**For advanced students!**

1. **Endless Mode** - Survive as long as possible
2. **Time Attack** - Kill 100 zombies as fast as possible
3. **One Life Mode** - Permadeath
4. **Melee Only** - No shooting allowed
5. **Pacifist** - Avoid all damage for 5 waves
6. **Speed Run** - Beat 10 waves in under 5 minutes
7. **Boss Rush** - Fight all bosses back-to-back
8. **Zombie Olympics** - Mini-games (target practice, speed trials)

---

### 22. Modding Support
**Let others customize!**

```cpp
// Load custom zombie types from file
void LoadCustomZombies(const char* filename)
{
    // Read JSON/TXT file
    // Parse zombie stats
    // Add to factory
}

// Custom wave definitions
void LoadCustomWaves(const char* filename)
{
    // Define spawn patterns
    // Custom difficulty curves
}
```

---

## 📚 Resources

**Free Assets:**
- OpenGameArt.org - sprites, sounds, music
- Kenney.nl - huge asset packs (CC0)
- Freesound.org - sound effects
- Incompetech.com - royalty-free music

**Raylib Resources:**
- raylib.com/examples.html - code examples
- github.com/raysan5/raylib - source code
- raylib Discord - community help

**Game Feel:**
- *The Art of Screenshake* by Jan Willem Nijman (YouTube)
- *Juice It or Lose It* (GDC talk)

---

## 🎁 Bonus Project Ideas

After Zombie Horde, try:

1. **Tower Defense** (Strategy, Factory, Object Pool)
2. **Roguelike** (Procedural generation, State, Component)
3. **Fighting Game** (Command for input, State for animations)
4. **Racing Game** (Strategy for AI, Component for vehicles)
5. **Platformer** (State for player states, Observer for events)

---

## 🎉 Final Tips

**Make it YOUR game:**
- Change theme: Zombies → Aliens, Robots, Knights
- Add unique mechanics: Grappling hook, Time control, Wall jumping
- Create original art
- Compose original music

**Share your work:**
- Upload to itch.io
- Post on Reddit (r/gamedev, r/raylib)
- Make a DevLog video
- Add to portfolio

**Keep learning:**
- Read "Game Programming Patterns" by Robert Nystrom
- Study open-source games
- Join game jams (Ludum Dare, GMTK)
- Build more games!

---

**Remember: The best way to learn is to BUILD!** 🚀