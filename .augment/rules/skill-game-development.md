---
type: agent_requested
description: Game development patterns: ECS architecture, object pooling, state machines, observer events, command pattern, and Unity/Godot/Unreal performance optimization.
---

# Game Development

## When to Use
Activate when building:
- Game systems in Unity, Unreal, Godot, or custom engines
- Game loop, ECS, or component-based architectures
- AI state machines, input handling, event systems
- Performance-critical game code (draw calls, GC pressure, physics)
- Object pooling, asset loading, or scene management systems

## Game Loop Architecture

```
while (running) {
    ProcessInput();          // Read player/network input
    Update(deltaTime);       // Physics, AI, animations, game logic
    LateUpdate();            // Camera follow, post-physics corrections
    Render();                // Submit draw calls to GPU
    Sleep(targetFrameTime);  // Frame rate cap
}
```

**Fixed vs variable timestep:**
- `FixedUpdate` / physics tick at fixed 50 Hz for determinism
- `Update` * `deltaTime` for visual smoothness
- Interpolate render positions between physics ticks for smooth display

## Entity Component System (ECS)

```csharp
// Component = pure data, no logic
public struct Position { public float X, Y, Z; }
public struct Velocity { public float X, Y, Z; }
public struct Health   { public int Current, Max; }

// Entity = just an ID
public struct Entity   { public int Id; }

// System = logic only, operates on component arrays (cache-friendly)
public class MovementSystem {
    public void Update(float dt, Span<Position> pos, Span<Velocity> vel) {
        for (int i = 0; i < pos.Length; i++) {
            pos[i].X += vel[i].X * dt;
            pos[i].Y += vel[i].Y * dt;
        }
    }
}
```

**Benefits:** Cache-friendly iteration (SoA layout), clear data/logic separation, easy parallelization.
**Unity DOTS:** Use `IComponentData`, `SystemBase`, `EntityCommandBuffer` for safe structural changes.

## Design Patterns

### Object Pool (zero runtime allocations)
```csharp
public class ObjectPool<T> where T : class, new() {
    private readonly Stack<T> _pool = new();
    private readonly Action<T> _reset;
    public ObjectPool(Action<T> reset, int warmup = 20) {
        _reset = reset;
        for (int i = 0; i < warmup; i++) _pool.Push(new T());
    }
    public T Get() => _pool.Count > 0 ? _pool.Pop() : new T();
    public void Return(T obj) { _reset(obj); _pool.Push(obj); }
}
// Pre-warm in Awake/Start. Never Instantiate/Destroy bullets at runtime.
```

### State Machine (enemy AI)
```csharp
public interface IState { void Enter(); void Update(float dt); void Exit(); }
public class StateMachine {
    private IState _current;
    public void ChangeState(IState next) {
        _current?.Exit(); _current = next; _current?.Enter();
    }
    public void Update(float dt) => _current?.Update(dt);
}
// States: IdleState → (player in range) → ChaseState → (in attack range) → AttackState
// Each state checks its own transition conditions in Update()
```

### Observer / Event Hub
```csharp
public class GameEvent<T> {
    private event Action<T> _listeners;
    public void Subscribe(Action<T> fn)   => _listeners += fn;
    public void Unsubscribe(Action<T> fn) => _listeners -= fn;
    public void Trigger(T data)           => _listeners?.Invoke(data);
}
public static class GameEvents {
    public static readonly GameEvent<int>   OnScoreChanged  = new();
    public static readonly GameEvent<float> OnHealthChanged = new();
    public static readonly GameEvent<string> OnGameOver     = new();
}
// Always call Unsubscribe in OnDisable/OnDestroy to prevent memory leaks.
```

### Command Pattern (input replay / undo)
```csharp
public interface ICommand { void Execute(); void Undo(); }
public class InputHandler {
    private readonly Stack<ICommand> _history = new();
    public void Run(ICommand cmd) { cmd.Execute(); _history.Push(cmd); }
    public void Undo() { if (_history.Count > 0) _history.Pop().Undo(); }
}
```

## Performance Patterns

### Memory / GC (zero allocations per frame)
- Pre-allocate: cache `WaitForSeconds`, use `StringBuilder`, avoid LINQ in Update
- Prefer structs for small value types in hot paths
- Pool all frequently spawned objects (bullets, particles, enemies, UI elements)

### Rendering
```csharp
// Static batching: mark objects Static in Inspector, or:
StaticBatchingUtility.Combine(staticObjects, rootObject);

// GPU instancing for many identical meshes (same material required)
Graphics.DrawMeshInstanced(mesh, 0, material, matrices);

// Share materials: use sharedMaterial (material creates an instance → breaks batching)
renderer.sharedMaterial.color = teamColor;
```

### LOD Setup
```csharp
LODGroup lod = go.AddComponent<LODGroup>();
lod.SetLODs(new[] {
    new LOD(0.6f, highDetailRenderers),    // 0–60% screen height
    new LOD(0.3f, mediumDetailRenderers),  // 60–30%
    new LOD(0.1f, lowDetailRenderers),     // 30–10%
});
lod.RecalculateBounds();
```

### Update Loop Optimization
```csharp
// Stagger expensive updates across frames
void Update() {
    if ((Time.frameCount + _staggerOffset) % 5 == 0) RunExpensiveAI();
}

// Distance-based update rate
float dist = Vector3.Distance(transform.position, player.position);
_updateInterval = dist < 10f ? 0.05f : dist < 50f ? 0.1f : 0.5f;
```

### Physics
- Prefer sphere/box colliders over mesh colliders (10–100× faster)
- Use Layer Collision Matrix to skip unnecessary collision pairs
- Set `sleepThreshold` to allow Rigidbodies to sleep
- Throttle raycasts with interval + layer mask filter

### Async Loading
```csharp
IEnumerator LoadSceneAsync(string name) {
    var op = SceneManager.LoadSceneAsync(name);
    op.allowSceneActivation = false;
    while (op.progress < 0.9f) yield return null;
    yield return new WaitForSeconds(0.5f); // show loading screen / fade
    op.allowSceneActivation = true;
}
```

## Performance Budget (60 FPS = 16.67 ms/frame)

| Budget Item | Target |
|-------------|--------|
| Game logic + scripts | 5–7 ms |
| Rendering (CPU side) | 3–5 ms |
| Physics | 2–3 ms |
| Reserve (GC, audio) | 2 ms |

**Profile first** with Unity Profiler / Unreal Insights / Godot Debugger before optimizing. Never guess.

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/game-developer
- Unity DOTS ECS: https://docs.unity3d.com/Packages/com.unity.entities@latest
- Unreal performance guide: https://docs.unrealengine.com/performance-guidelines

