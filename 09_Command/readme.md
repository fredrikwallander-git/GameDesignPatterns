# Command Pattern

## The Problem

Your **Zombie Horde** handles input directly in the update loop:

```cpp
void Player::Update(float deltaTime)
{
    // Movement
    if (IsKeyDown(KEY_W)) position.y -= speed * deltaTime;
    if (IsKeyDown(KEY_S)) position.y += speed * deltaTime;
    if (IsKeyDown(KEY_A)) position.x -= speed * deltaTime;
    if (IsKeyDown(KEY_D)) position.x += speed * deltaTime;
    
    // Shooting
    if (IsKeyPressed(KEY_SPACE))
    {
        Vector2 direction = CalculateShootDirection();
        bulletPool.Spawn(position, direction, 500.0f);
    }
    
    // Dash
    if (IsKeyPressed(KEY_LEFT_SHIFT))
    {
        velocity = CalculateDashDirection() * dashSpeed;
        dashCooldown = 2.0f;
    }
}
```

### What's Wrong?

**Problem 1: Hardcoded keys**
```cpp
// Want to rebind keys?
// Change KEY_W to KEY_UP?
// Now you need to modify Player.cpp!
```

**Problem 2: Can't record/replay**
```cpp
// Want to record gameplay for replay?
// How do you capture "W was pressed"?
// Input is directly tied to execution!
```

**Problem 3: No undo**
```cpp
// Made a mistake moving?
// Can't undo - the action already happened!
```

**Problem 4: Can't queue actions**
```cpp
// Want to buffer inputs during lag?
// "Player pressed shoot, then dash"
// Can't store this - actions execute immediately!
```

**Problem 5: Controller vs Keyboard**
```cpp
// Want gamepad support?
// Now you need:
if (IsKeyDown(KEY_W) || IsGamepadButtonDown(GAMEPAD_DPAD_UP))
// Duplicated logic everywhere!
```

**Problem 6: AI can't use player actions**
```cpp
// Want AI to "dash" like player?
// Copy-paste the dash code?
// Now you have duplicate dash logic!
```

---

## The Solution: Command Pattern

**Encapsulate actions as objects that can be stored, queued, and replayed.**

Think of it like **recording a macro**:
- Each button press creates a Command object
- Commands can be saved to a list
- Replay the list = replay the gameplay
- Same commands work for player AND AI

### Key Characteristics:
1. **Actions are objects** (MoveCommand, ShootCommand, etc.)
2. **Can be stored** - save to list, file, network packet
3. **Can be undone** - Command has Undo() method
4. **Can be replayed** - Execute commands from history
5. **Decoupled** - input separated from execution

---

## Implementation in Zombie Horde

### Command.h (Base Class)

```cpp
#ifndef COMMAND_H
#define COMMAND_H

class Entity;  // Forward declaration

// Base command class
class Command
{
public:
    virtual ~Command() = default;
    
    // Execute the command
    virtual void Execute(Entity* entity, float deltaTime) = 0;
    
    // Undo the command (optional - for undo/redo systems)
    virtual void Undo(Entity* entity) {}
    
    // Can this command be undone?
    virtual bool IsUndoable() const { return false; }
    
    // Get command name (for debugging/display)
    virtual const char* GetName() const = 0;
};

#endif
```

---

### MoveCommand.h

```cpp
#ifndef MOVE_COMMAND_H
#define MOVE_COMMAND_H

#include "Command.h"
#include "raylib.h"

class MoveCommand : public Command
{
private:
    Vector2 direction;
    float speed;
    Vector2 previousPosition;  // For undo
    
public:
    MoveCommand(Vector2 dir, float moveSpeed)
        : direction(dir)
        , speed(moveSpeed)
        , previousPosition({0, 0})
    {}
    
    void Execute(Entity* entity, float deltaTime) override
    {
        // Store previous position for undo
        previousPosition = entity->GetPosition();
        
        // Move entity
        Vector2 newPosition = {
            previousPosition.x + direction.x * speed * deltaTime,
            previousPosition.y + direction.y * speed * deltaTime
        };
        
        entity->SetPosition(newPosition);
    }
    
    void Undo(Entity* entity) override
    {
        entity->SetPosition(previousPosition);
    }
    
    bool IsUndoable() const override { return true; }
    
    const char* GetName() const override { return "Move"; }
};

#endif
```

---

### ShootCommand.h

```cpp
#ifndef SHOOT_COMMAND_H
#define SHOOT_COMMAND_H

#include "Command.h"
#include "BulletPool.h"
#include "raylib.h"

class ShootCommand : public Command
{
private:
    Vector2 direction;
    float bulletSpeed;
    BulletPool* bulletPool;
    
public:
    ShootCommand(Vector2 dir, float speed, BulletPool* pool)
        : direction(dir)
        , bulletSpeed(speed)
        , bulletPool(pool)
    {}
    
    void Execute(Entity* entity, float deltaTime) override
    {
        // Spawn bullet from entity position
        Vector2 position = entity->GetPosition();
        bulletPool->Spawn(position, direction, bulletSpeed);
    }
    
    // Can't undo shooting - bullet already fired!
    bool IsUndoable() const override { return false; }
    
    const char* GetName() const override { return "Shoot"; }
};

#endif
```

---

### DashCommand.h

```cpp
#ifndef DASH_COMMAND_H
#define DASH_COMMAND_H

#include "Command.h"
#include "raylib.h"

class DashCommand : public Command
{
private:
    Vector2 direction;
    float dashDistance;
    float dashSpeed;
    Vector2 previousPosition;
    
public:
    DashCommand(Vector2 dir, float distance = 100.0f, float speed = 500.0f)
        : direction(dir)
        , dashDistance(distance)
        , dashSpeed(speed)
        , previousPosition({0, 0})
    {}
    
    void Execute(Entity* entity, float deltaTime) override
    {
        previousPosition = entity->GetPosition();
        
        // Instant dash to new position
        Vector2 newPosition = {
            previousPosition.x + direction.x * dashDistance,
            previousPosition.y + direction.y * dashDistance
        };
        
        entity->SetPosition(newPosition);
        
        // Could also set velocity for smooth dash:
        // entity->SetVelocity(direction.x * dashSpeed, direction.y * dashSpeed);
    }
    
    void Undo(Entity* entity) override
    {
        entity->SetPosition(previousPosition);
    }
    
    bool IsUndoable() const override { return true; }
    
    const char* GetName() const override { return "Dash"; }
};

#endif
```

---

### InputHandler.h

```cpp
#ifndef INPUT_HANDLER_H
#define INPUT_HANDLER_H

#include "Command.h"
#include "raylib.h"
#include <map>

class InputHandler
{
private:
    // Key bindings: Key → Command
    std::map<int, Command*> keyBindings;
    
public:
    ~InputHandler()
    {
        // Clean up commands
        for (auto& pair : keyBindings)
        {
            delete pair.second;
        }
    }
    
    // Bind a key to a command
    void BindKey(int key, Command* command)
    {
        // Delete old binding if exists
        auto it = keyBindings.find(key);
        if (it != keyBindings.end())
        {
            delete it->second;
        }
        
        keyBindings[key] = command;
    }
    
    // Handle input and return command if key pressed
    Command* HandleInput()
    {
        for (auto& pair : keyBindings)
        {
            if (IsKeyDown(pair.first))
            {
                return pair.second;
            }
        }
        
        return nullptr;  // No command
    }
    
    // Get command for specific key
    Command* GetCommand(int key)
    {
        auto it = keyBindings.find(key);
        if (it != keyBindings.end())
        {
            return it->second;
        }
        return nullptr;
    }
};

#endif
```

---

### CommandQueue.h (For Buffering/Replay)

```cpp
#ifndef COMMAND_QUEUE_H
#define COMMAND_QUEUE_H

#include "Command.h"
#include <vector>
#include <memory>

class CommandQueue
{
private:
    std::vector<std::unique_ptr<Command>> commands;
    
public:
    // Add command to queue
    void Enqueue(Command* command)
    {
        commands.push_back(std::unique_ptr<Command>(command));
    }
    
    // Execute all commands
    void ExecuteAll(Entity* entity, float deltaTime)
    {
        for (auto& command : commands)
        {
            command->Execute(entity, deltaTime);
        }
        commands.clear();
    }
    
    // Get queue size
    int Size() const { return commands.size(); }
    
    // Clear queue without executing
    void Clear()
    {
        commands.clear();
    }
};

#endif
```

---

### CommandHistory.h (For Undo/Redo)

```cpp
#ifndef COMMAND_HISTORY_H
#define COMMAND_HISTORY_H

#include "Command.h"
#include <vector>
#include <memory>

class CommandHistory
{
private:
    std::vector<std::unique_ptr<Command>> history;
    int currentIndex;
    int maxHistorySize;
    
public:
    CommandHistory(int maxSize = 100)
        : currentIndex(-1)
        , maxHistorySize(maxSize)
    {}
    
    // Execute and record command
    void ExecuteCommand(Command* command, Entity* entity, float deltaTime)
    {
        // Execute the command
        command->Execute(entity, deltaTime);
        
        // Only record undoable commands
        if (command->IsUndoable())
        {
            // Remove any commands after current index (redo branch)
            if (currentIndex < (int)history.size() - 1)
            {
                history.erase(history.begin() + currentIndex + 1, history.end());
            }
            
            // Add to history
            history.push_back(std::unique_ptr<Command>(command));
            currentIndex++;
            
            // Limit history size
            if ((int)history.size() > maxHistorySize)
            {
                history.erase(history.begin());
                currentIndex--;
            }
        }
        else
        {
            // Command not undoable, don't store it
            delete command;
        }
    }
    
    // Undo last command
    bool Undo(Entity* entity)
    {
        if (currentIndex >= 0)
        {
            history[currentIndex]->Undo(entity);
            currentIndex--;
            return true;
        }
        return false;
    }
    
    // Redo next command
    bool Redo(Entity* entity, float deltaTime)
    {
        if (currentIndex < (int)history.size() - 1)
        {
            currentIndex++;
            history[currentIndex]->Execute(entity, deltaTime);
            return true;
        }
        return false;
    }
    
    // Check if can undo/redo
    bool CanUndo() const { return currentIndex >= 0; }
    bool CanRedo() const { return currentIndex < (int)history.size() - 1; }
    
    // Get history size
    int GetHistorySize() const { return history.size(); }
    
    // Clear history
    void Clear()
    {
        history.clear();
        currentIndex = -1;
    }
};

#endif
```

---

### Recorder.h (For Replay System)

```cpp
#ifndef RECORDER_H
#define RECORDER_H

#include "Command.h"
#include <vector>
#include <memory>
#include <fstream>

struct RecordedCommand
{
    std::unique_ptr<Command> command;
    float timestamp;
};

class Recorder
{
private:
    std::vector<RecordedCommand> recording;
    bool isRecording;
    bool isReplaying;
    float recordingTime;
    float replayTime;
    int replayIndex;
    
public:
    Recorder()
        : isRecording(false)
        , isReplaying(false)
        , recordingTime(0.0f)
        , replayTime(0.0f)
        , replayIndex(0)
    {}
    
    // Start recording
    void StartRecording()
    {
        recording.clear();
        isRecording = true;
        recordingTime = 0.0f;
    }
    
    // Stop recording
    void StopRecording()
    {
        isRecording = false;
    }
    
    // Record a command
    void RecordCommand(Command* command, float deltaTime)
    {
        if (!isRecording) return;
        
        recordingTime += deltaTime;
        
        RecordedCommand rec;
        rec.command = std::unique_ptr<Command>(command);
        rec.timestamp = recordingTime;
        
        recording.push_back(std::move(rec));
    }
    
    // Start replaying
    void StartReplay()
    {
        isReplaying = true;
        replayTime = 0.0f;
        replayIndex = 0;
    }
    
    // Stop replaying
    void StopReplay()
    {
        isReplaying = false;
    }
    
    // Update replay
    void UpdateReplay(Entity* entity, float deltaTime)
    {
        if (!isReplaying) return;
        
        replayTime += deltaTime;
        
        // Execute all commands that should happen by now
        while (replayIndex < (int)recording.size() &&
               recording[replayIndex].timestamp <= replayTime)
        {
            recording[replayIndex].command->Execute(entity, deltaTime);
            replayIndex++;
        }
        
        // Check if replay finished
        if (replayIndex >= (int)recording.size())
        {
            StopReplay();
        }
    }
    
    // Save recording to file
    void SaveToFile(const char* filename)
    {
        // Implementation would serialize commands to file
        // For now, just a placeholder
        std::cout << "Saving " << recording.size() << " commands to " << filename << std::endl;
    }
    
    // Load recording from file
    void LoadFromFile(const char* filename)
    {
        // Implementation would deserialize commands from file
        std::cout << "Loading recording from " << filename << std::endl;
    }
    
    // Status
    bool IsRecording() const { return isRecording; }
    bool IsReplaying() const { return isReplaying; }
    int GetRecordedCommandCount() const { return recording.size(); }
};

#endif
```

---

## How to Use It

### Basic Usage - Input Handling

```cpp
// Create input handler
InputHandler inputHandler;
BulletPool bulletPool(100);

// Bind keys to commands
inputHandler.BindKey(KEY_W, new MoveCommand({0, -1}, 200.0f));
inputHandler.BindKey(KEY_S, new MoveCommand({0, 1}, 200.0f));
inputHandler.BindKey(KEY_A, new MoveCommand({-1, 0}, 200.0f));
inputHandler.BindKey(KEY_D, new MoveCommand({1, 0}, 200.0f));
inputHandler.BindKey(KEY_SPACE, new ShootCommand({1, 0}, 500.0f, &bulletPool));
inputHandler.BindKey(KEY_LEFT_SHIFT, new DashCommand({1, 0}, 100.0f));

// In game loop
Command* command = inputHandler.HandleInput();
if (command)
{
    command->Execute(player, deltaTime);
}
```

### With Command History (Undo/Redo)

```cpp
CommandHistory history(50);  // Keep last 50 commands

// Execute command with history
Command* command = inputHandler.HandleInput();
if (command)
{
    history.ExecuteCommand(command, player, deltaTime);
}

// Undo with 'Z' key
if (IsKeyPressed(KEY_Z) && history.CanUndo())
{
    history.Undo(player);
}

// Redo with 'Y' key
if (IsKeyPressed(KEY_Y) && history.CanRedo())
{
    history.Redo(player, deltaTime);
}
```

### Recording and Replay

```cpp
Recorder recorder;

// Start recording with 'R'
if (IsKeyPressed(KEY_R))
{
    if (!recorder.IsRecording())
    {
        recorder.StartRecording();
        std::cout << "Recording started!" << std::endl;
    }
    else
    {
        recorder.StopRecording();
        std::cout << "Recording stopped!" << std::endl;
    }
}

// During gameplay
Command* command = inputHandler.HandleInput();
if (command)
{
    command->Execute(player, deltaTime);
    recorder.RecordCommand(command, deltaTime);
}

// Replay with 'P'
if (IsKeyPressed(KEY_P))
{
    recorder.StartReplay();
}

// Update replay
if (recorder.IsReplaying())
{
    recorder.UpdateReplay(player, deltaTime);
}
```

### AI Using Player Commands

```cpp
class ZombieAI
{
    void Update(Vector2 playerPos)
    {
        // AI can use same commands as player!
        Vector2 direction = Normalize(playerPos - position);
        
        // Create and execute move command
        MoveCommand moveCmd(direction, 80.0f);
        moveCmd.Execute(zombieEntity, deltaTime);
        
        // If close enough, dash attack!
        if (Distance(position, playerPos) < 100.0f)
        {
            DashCommand dashCmd(direction, 50.0f, 300.0f);
            dashCmd.Execute(zombieEntity, deltaTime);
        }
    }
};
```

---

## How It Works

### Before Command Pattern (Direct Input)

```
Input → Execute Immediately
  ↓
Player.Update()
{
    if (KEY_W) position.y -= speed;  // Hardcoded, immediate
    if (KEY_SPACE) Shoot();          // Can't record
}

❌ Can't rebind keys
❌ Can't record/replay
❌ Can't undo
❌ Can't queue
```

### After Command Pattern (Decoupled)

```
Input → Create Command → Store/Execute/Queue/Record
         ↓
    Command Object
    - Can be stored in list
    - Can be saved to file
    - Can be undone
    - Can be replayed
    - Can be executed by anyone (player, AI, replay)

✅ Rebind keys: just change binding
✅ Record/replay: store commands with timestamps
✅ Undo: call command.Undo()
✅ Queue: add to list, execute later
```

---

## Exercise 1: Basic Extension - Rebindable Controls

**Goal:** Create a UI to let players rebind keys.

**Requirements:**
1. Display current key bindings
2. Click on action to rebind
3. Press new key to bind
4. Save bindings to file
5. Load bindings on startup

**Starter code:**
```cpp
class ControlsMenu
{
private:
    InputHandler* inputHandler;
    int selectedAction;  // -1 = none, 0 = move up, 1 = move down, etc.
    bool waitingForKey;
    
    struct ActionBinding
    {
        const char* name;
        int currentKey;
        Command* command;
    };
    
    std::vector<ActionBinding> bindings;
    
public:
    void Draw()
    {
        DrawText("CONTROLS", 100, 50, 40, WHITE);
        
        for (int i = 0; i < bindings.size(); i++)
        {
            // Draw action name
            DrawText(bindings[i].name, 100, 100 + i * 40, 20, WHITE);
            
            // Draw current key
            const char* keyName = GetKeyName(bindings[i].currentKey);
            Color color = (selectedAction == i) ? YELLOW : GRAY;
            DrawText(keyName, 300, 100 + i * 40, 20, color);
            
            if (selectedAction == i && waitingForKey)
            {
                DrawText("Press new key...", 450, 100 + i * 40, 20, GREEN);
            }
        }
    }
    
    void Update()
    {
        // Check for mouse clicks on actions
        // If clicked, wait for key press
        // When key pressed, rebind
        
        if (waitingForKey)
        {
            int key = GetKeyPressed();
            if (key != 0)
            {
                // Rebind
                inputHandler->BindKey(key, bindings[selectedAction].command);
                bindings[selectedAction].currentKey = key;
                waitingForKey = false;
            }
        }
    }
    
    void SaveBindings(const char* filename)
    {
        // Save to file
        std::ofstream file(filename);
        for (auto& binding : bindings)
        {
            file << binding.name << " " << binding.currentKey << std::endl;
        }
    }
    
    void LoadBindings(const char* filename)
    {
        // Load from file
    }
};
```

---

## Exercise 2: Intermediate Challenge - Macro System

**Goal:** Record a sequence of commands and replay with one keypress.

**Example:**
- Record: Move right, Shoot, Dash, Shoot
- Save as "Combo 1"
- Press F1 to execute entire sequence

**Requirements:**
1. Record multiple commands in sequence
2. Store as named macro
3. Bind macro to hotkey (F1-F12)
4. Execute all commands in sequence when hotkey pressed
5. Save/load macros to file

**Starter code:**
```cpp
class MacroCommand : public Command
{
private:
    std::vector<std::unique_ptr<Command>> commands;
    std::string name;
    
public:
    MacroCommand(const std::string& macroName)
        : name(macroName)
    {}
    
    void AddCommand(Command* command)
    {
        commands.push_back(std::unique_ptr<Command>(command));
    }
    
    void Execute(Entity* entity, float deltaTime) override
    {
        // Execute all commands in sequence
        for (auto& command : commands)
        {
            command->Execute(entity, deltaTime);
        }
    }
    
    const char* GetName() const override { return name.c_str(); }
};

// Usage:
MacroCommand* combo1 = new MacroCommand("Dash Attack Combo");
combo1->AddCommand(new DashCommand({1, 0}, 50.0f));
combo1->AddCommand(new ShootCommand({1, 0}, 500.0f, &bulletPool));
combo1->AddCommand(new ShootCommand({1, 0}, 500.0f, &bulletPool));

inputHandler.BindKey(KEY_F1, combo1);
```

---

## Exercise 3: Advanced - Network Replay

**Goal:** Send commands over network for multiplayer.

**Concept:**
- Client records commands with timestamps
- Send commands to server
- Server executes commands
- Other clients receive and replay commands

**Requirements:**
1. Serialize commands to byte array
2. Send over network (simulate with file I/O)
3. Deserialize and execute on receiver
4. Handle latency (commands arrive late)

**Starter code:**
```cpp
struct SerializedCommand
{
    int commandType;  // 0=Move, 1=Shoot, 2=Dash
    float timestamp;
    float data[4];    // Command-specific data
};

class CommandSerializer
{
public:
    static std::vector<byte> Serialize(Command* command, float timestamp)
    {
        // Convert command to bytes
        std::vector<byte> data;
        // ... implementation
        return data;
    }
    
    static Command* Deserialize(const std::vector<byte>& data)
    {
        // Convert bytes to command
        // ... implementation
        return nullptr;
    }
};
```

---

## Exercise 4: Advanced (Optional) - Time Rewind

**Goal:** Implement a "time rewind" feature like in Braid or Prince of Persia.

**Requirements:**
1. Record game state + commands every frame
2. When player presses 'Rewind':
    - Stop normal gameplay
    - Play recording backwards
    - Restore previous game states
3. When player releases 'Rewind':
    - Resume from that point

**This is VERY advanced!** Requires:
- Storing entire game state (positions, health, etc.)
- Interpolating between states
- Handling what happens to bullets/zombies during rewind

---

## Common Mistakes

### ❌ Mistake 1: Commands depending on current state

```cpp
// WRONG - command checks if can execute
class ShootCommand : public Command
{
    void Execute(Entity* entity, float deltaTime) override
    {
        if (entity->HasAmmo())  // BAD! Command shouldn't check this
        {
            Shoot();
        }
    }
};
```

**Fix:** Check before creating command
```cpp
// CORRECT - check before creating command
if (player->HasAmmo())
{
    Command* cmd = new ShootCommand(...);
    cmd->Execute(player, deltaTime);
}
```

---

### ❌ Mistake 2: Commands storing entity reference

```cpp
// WRONG - storing entity pointer
class MoveCommand : public Command
{
    Entity* entity;  // BAD! What if entity is deleted?
    
    MoveCommand(Entity* e) : entity(e) {}
    
    void Execute(Entity* unused, float dt) override
    {
        entity->Move();  // Could be dangling pointer!
    }
};
```

**Fix:** Pass entity when executing
```cpp
// CORRECT - entity passed at execution time
void Execute(Entity* entity, float dt) override
{
    entity->Move();  // Safe!
}
```

---

### ❌ Mistake 3: Not handling deltaTime properly in replay

```cpp
// WRONG - using deltaTime in recorded commands
recorder.RecordCommand(new MoveCommand(dir, speed), deltaTime);
// deltaTime changes each frame - replay will be wrong!
```

**Fix:** Record fixed timestep or compensate
```cpp
// CORRECT - store actual timestamp
void RecordCommand(Command* cmd, float timestamp);
```

---

### ❌ Mistake 4: Memory leaks with command creation

```cpp
// WRONG - creating commands every frame
void Update()
{
    Command* cmd = new MoveCommand(...);  // Memory leak!
    cmd->Execute(player, deltaTime);
    // Never deleted!
}
```

**Fix:** Reuse commands or use smart pointers
```cpp
// CORRECT - reuse command
MoveCommand moveUpCmd({0, -1}, 200.0f);

void Update()
{
    if (IsKeyDown(KEY_W))
    {
        moveUpCmd.Execute(player, deltaTime);  // Reuse!
    }
}
```

---

### ❌ Mistake 5: Undoing non-undoable commands

```cpp
// WRONG - trying to undo shooting
history.Undo(player);  // Undoes dash, then tries to undo shoot - impossible!
```

**Fix:** Check IsUndoable()
```cpp
// CORRECT
bool CommandHistory::Undo(Entity* entity)
{
    while (currentIndex >= 0)
    {
        if (history[currentIndex]->IsUndoable())
        {
            history[currentIndex]->Undo(entity);
            currentIndex--;
            return true;
        }
        currentIndex--;  // Skip non-undoable commands
    }
    return false;
}
```

---

## When to Use Command Pattern

### ✅ Good Use Cases:
- **Input handling** (rebindable controls)
- **Undo/redo systems** (editors, strategy games)
- **Replay systems** (demos, spectator mode)
- **Macro systems** (combo attacks, custom controls)
- **Network multiplayer** (send commands, not state)
- **AI** (reuse player commands for enemies)
- **Tutorial systems** (replay recorded actions)

### ❌ Bad Use Cases:
- **Simple, unchanging controls** (if keys never change, just use if statements)
- **No replay/undo needed** (adds unnecessary complexity)
- **High-frequency actions** (moving mouse - too many commands)
- **Stateless actions** (playing a sound - no need to command-ify)

### The Rule of Thumb:
**"If you need to store, queue, undo, or replay actions, use Command Pattern."**

---

## Command Pattern Variations

### 1. Simple Commands (What we built)
```cpp
class Command
{
    virtual void Execute(Entity* entity) = 0;
};
```
- Clean, flexible
- Good for most games

### 2. Functional Commands
```cpp
using Command = std::function<void()>;

Command moveUp = []() { player.Move(0, -1); };
```
- Very simple
- Hard to undo
- C++ lambdas

### 3. Parameterized Commands
```cpp
class Command
{
    virtual void Execute(Entity* entity, CommandParams params) = 0;
};
```
- More flexible
- Can pass different data each time

---

## Summary

✅ **Command Pattern** encapsulates actions as objects<br>
✅ **Rebindable controls** - change key bindings easily<br>
✅ **Undo/Redo** - store and reverse commands<br>
✅ **Replay** - record and playback gameplay<br>
✅ **Queuing** - buffer actions for later execution<br>
✅ **Reusable** - same commands for player, AI, replay

**The pattern you just learned:**
```cpp
// 1. Define command interface
class Command
{
    virtual void Execute(Entity* entity, float dt) = 0;
    virtual void Undo(Entity* entity) {}
};

// 2. Create concrete commands
class MoveCommand : public Command { /* ... */ };
class ShootCommand : public Command { /* ... */ };

// 3. Use commands
Command* cmd = inputHandler.HandleInput();
if (cmd) cmd->Execute(player, deltaTime);
```

**Before Command:**
```cpp
void Player::Update()
{
    if (IsKeyDown(KEY_W)) position.y -= speed * deltaTime;
    if (IsKeyDown(KEY_SPACE)) Shoot();
    
    // ❌ Hardcoded keys
    // ❌ Can't record/replay
    // ❌ Can't undo
    // ❌ Can't rebind
}
```

**After Command:**
```cpp
Command* cmd = inputHandler.HandleInput();
if (cmd)
{
    history.ExecuteCommand(cmd, player, deltaTime);
    recorder.RecordCommand(cmd, deltaTime);
}

// ✅ Rebindable: inputHandler.BindKey(newKey, command)
// ✅ Recordable: recorder.StartRecording()
// ✅ Undoable: history.Undo(player)
// ✅ Replayable: recorder.StartReplay()
```