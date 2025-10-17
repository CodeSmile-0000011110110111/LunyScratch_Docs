# 2025-10-17_.md

Godot will likely require a bit more scaffolding and maintenance. 

It's limited in what callbacks it offers, for instance there's no straightforward hooks for "scene loaded" and "scene unloaded". 

It doesn't have a LateUpdate nor a clear way to call "OnDestroy". It wasn't necessary to have this in the proof of concept but might reveal themselves later. I imagine I'll have to make all scratch objects register and unregister with the runtime in order to ensure destruction, and to call LateUpdate after all scratches' Update (process) ran.

Godot also doesn't provide direct input events for "KeyJustPressed/Released" without using Actions. I regard Action Map support as higher priority because of this. But also requiring direct keystroke tests to be on a tick system so that each key has a flag (in a bitfield) whether it was "pressed/released this frame". 

Camera is really dumb. There are no tracking/follow/lookat targets. That warrants a simple wrapper, and a way to toggle between cameras. I'll definitely check out the Cinemachine clone extension ([Phantom Camera](https://github.com/ramokz/phantom-camera)) which may be an option to integrate.

The 2D/3D split is real, down to every node. So I'll effectively have to create many Blocks in 2D and 3D variants.

I should also consider adding more safety checks as so many things are non-obvious, like forgetting to enable "Contact Monitor" and setting max contacts to > 0, otherwise there'll be no collision events. Similar issue if the root node isn't a Rigidbody, in which case the root will never move away from its instanced position. Thus I had to create a ScratchRigidbody3D base class, and I may even need to make one for each common base type (3x Node, 2x Rigidbody, Control, ..).

Collision shapes, according to warnings and initial experiences (didn't verify), appear to not support scaling. That means we need to modify the scale of a mesh and the size of the collision shape separately. I've also based my experience solely on Jolt physics, not the default one, which may have different limitations.

On top, integration with GDScript would be "this engine only" if I'd go through with it. I may just ignore that since there's no editor integration need for scratches anyway, so Lua should be fine.

## Open Issues

- Turning left/right all by its own is not interpolated, jitters, slows down. It's smooth when the linear velocity is non-zero.


## Avoid subclassing?

Excellent question — and you’ve run into one of the core architectural friction points when trying to build cross-cutting frameworks in Godot (especially in C#). Let’s unpack what’s happening and what your realistic options are.

---

## 🧩 The Problem You’re Facing

You want to:

* Run your own code for every node’s lifecycle events (`_Ready()`, `_Process()`, `_PhysicsProcess()`, etc.)
* Without subclassing every built-in Godot node type (`Node2D`, `Node3D`, `RigidBody3D`, etc.)
* And ideally without forking / modifying the C++ engine.

That’s *exactly* the use case where Unity-style “component injection” or “global behavior hooks” would help — but Godot doesn’t have a public hook mechanism for that, because its event model is based on virtual methods called from C++ directly into your script.

---

## ⚙️ Option 1: A Universal “Lifecycle Manager” (Non-Subclass Hook)

You can attach a **single global autoload** (singleton) that connects to the `SceneTree`’s signals and walks the node tree, attaching handlers dynamically.

Here’s the core idea:

```csharp
// Autoloaded singleton: LifecycleManager.cs
using Godot;
using System;
using System.Collections.Generic;

public partial class LifecycleManager : Node
{
    public override void _Ready()
    {
        GetTree().NodeAdded += OnNodeAdded;
        GetTree().NodeRemoved += OnNodeRemoved;
    }

    private void OnNodeAdded(Node node)
    {
        if (node.IsInsideTree())
            HookNode(node);
    }

    private void HookNode(Node node)
    {
        // Example: connect ready and process events dynamically
        node.Ready += () => OnNodeReady(node);
        node.Process += (delta) => OnNodeProcess(node, delta);
    }

    private void OnNodeReady(Node node)
    {
        GD.Print($"[LunyScratch] Ready: {node.Name}");
        // Optionally: Register with your framework
    }

    private void OnNodeProcess(Node node, double delta)
    {
        // Execute LunyScratch per-node logic
    }

    private void OnNodeRemoved(Node node)
    {
        GD.Print($"[LunyScratch] Removed: {node.Name}");
    }
}
```

### ✅ Pros

* **No subclassing required.** Works with any node in any scene.
* **Completely runtime-driven.** You can inspect the node type, metadata, or groups and act accordingly.
* **Autoloaded** means you only need one instance for the entire project.

### ⚠️ Cons

* You can’t hook into `_Process()` via virtual dispatch — but you *can* use the `.Process` event introduced in Godot 4.x’s C# API.
* You need to manually check node types and attach custom behaviors.
* It’s global — so you must be careful about performance when many nodes are added/removed.

---

## 🧩 Option 2: “Behavior Components” via Groups or Scripts

You can emulate Unity-style components by defining your own “behavior” base class that can be added to any node as an attached script (not subclass).

```csharp
// LunyBehavior.cs
using Godot;

public partial class LunyBehavior : Node
{
    public virtual void OnReady(Node owner) {}
    public virtual void OnProcess(Node owner, double delta) {}
}
```

Then use your LifecycleManager from Option 1 to find nodes that have these behaviors attached:

```csharp
private void HookNode(Node node)
{
    foreach (var child in node.GetChildren())
    {
        if (child is LunyBehavior behavior)
        {
            node.Ready += () => behavior.OnReady(node);
            node.Process += (delta) => behavior.OnProcess(node, delta);
        }
    }
}
```

Now users can just drop your `LunyBehavior` components as children of existing nodes — no subclassing of Node3D, Node2D, etc.

---

## 🧠 Option 3: Use SceneTree Iteration + Reflection

If you want truly automatic behavior injection, you can periodically iterate over all nodes and use reflection to invoke methods (slower but flexible):

```csharp
public override void _Process(double delta)
{
    foreach (Node node in GetTree().GetNodesInGroup("LunyMonitored"))
    {
        var method = node.GetType().GetMethod("LunyProcess");
        method?.Invoke(node, new object[]{ delta });
    }
}
```

Users can then just define a `void LunyProcess(double delta)` method in *any* script.

This pattern trades type safety for ease of integration — quite similar to Unity’s “magic method” names.

---

## 🧩 Option 4: Editor-Side Tooling

You could also make an **EditorPlugin** that automatically adds a `LunyBridge` script to every node on scene load, or that injects your `LifecycleManager` script at runtime.
This is useful if you want full IDE-level control (like automatic wrapping or attribute-based hooks).

---

## 🧱 Summary of Viable Strategies

| Strategy                          | No Subclassing | Safe | Performance | Complexity | Notes                        |
| --------------------------------- | -------------- | ---- | ----------- | ---------- | ---------------------------- |
| Global `SceneTree` hook (signals) | ✅              | ✅    | ⚠️          | ⭐          | Best all-around, engine-safe |
| Behavior components               | ✅              | ✅    | ✅           | ⭐⭐         | Unity-style composition      |
| Reflection + naming convention    | ✅              | ⚠️   | ⚠️          | ⭐⭐⭐        | Dynamic, but less performant |
| Editor plugin auto-injection      | ✅              | ✅    | ✅           | ⭐⭐⭐⭐       | More setup, powerful         |

---

## 💡 Recommendation (for your use case)

Since your **LunyScratch** framework acts like a universal DSL runtime, the cleanest pattern is:

* ✅ Use an **autoloaded LifecycleManager** that hooks `NodeAdded` / `NodeRemoved`.
* ✅ Let users add `LunyBehavior` child nodes (or mark nodes via group or metadata).
* ✅ Optionally, use reflection or attributes (`[LunyHook]`) to auto-register per-node logic.

That gives you **engine-level integration** without subclassing *and* without modifying Godot’s source.

---

Would you like me to show a concrete implementation pattern combining **SceneTree hooks + attribute-based behaviors** (like Unity’s `[RequireComponent]` but in pure C# Godot)? It’s the most robust and scalable version of this idea.


----
Perfect 👌 — this pattern is the cleanest way to implement “engine-wide hooks” for Godot in **C#** without subclassing, modifying C++, or requiring users to use special node types.

Below is a full **attribute-driven, component-style integration** pattern that you can drop into your LunyScratch framework.

---

## 🧩 1. Define the Attribute(s)

You’ll define C# attributes that mark methods or properties that your framework should automatically call when node lifecycle events occur.

```csharp
// LunyAttributes.cs
using System;

[AttributeUsage(AttributeTargets.Method)]
public class LunyReadyAttribute : Attribute {}

[AttributeUsage(AttributeTargets.Method)]
public class LunyProcessAttribute : Attribute {}

[AttributeUsage(AttributeTargets.Method)]
public class LunyPhysicsAttribute : Attribute {}
```

These act like “opt-in” hooks, similar to Unity’s `Start()`, `Update()`, etc.

---

## 🧱 2. The Lifecycle Manager (Autoload Singleton)

Your global manager listens to the `SceneTree` and automatically hooks all nodes that define these attributes.

```csharp
// LunyLifecycleManager.cs
using Godot;
using System;
using System.Reflection;
using System.Collections.Generic;

public partial class LunyLifecycleManager : Node
{
    private readonly Dictionary<Node, List<Action<double>>> _processHooks = new();
    private readonly Dictionary<Node, List<Action<double>>> _physicsHooks = new();

    public override void _Ready()
    {
        GetTree().NodeAdded += OnNodeAdded;
        GetTree().NodeRemoved += OnNodeRemoved;
        GD.Print("[LunyScratch] LifecycleManager initialized.");
    }

    private void OnNodeAdded(Node node)
    {
        HookNode(node);
    }

    private void OnNodeRemoved(Node node)
    {
        _processHooks.Remove(node);
        _physicsHooks.Remove(node);
    }

    private void HookNode(Node node)
    {
        var type = node.GetType();

        // READY hooks
        foreach (var m in type.GetMethods(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic))
        {
            if (m.GetCustomAttribute<LunyReadyAttribute>() != null)
            {
                node.Ready += () => m.Invoke(node, null);
            }

            if (m.GetCustomAttribute<LunyProcessAttribute>() != null)
            {
                var act = (Action<double>)Delegate.CreateDelegate(typeof(Action<double>), node, m, false);
                if (act != null)
                {
                    if (!_processHooks.ContainsKey(node))
                        _processHooks[node] = new List<Action<double>>();
                    _processHooks[node].Add(act);
                }
            }

            if (m.GetCustomAttribute<LunyPhysicsAttribute>() != null)
            {
                var act = (Action<double>)Delegate.CreateDelegate(typeof(Action<double>), node, m, false);
                if (act != null)
                {
                    if (!_physicsHooks.ContainsKey(node))
                        _physicsHooks[node] = new List<Action<double>>();
                    _physicsHooks[node].Add(act);
                }
            }
        }
    }

    public override void _Process(double delta)
    {
        foreach (var (node, acts) in _processHooks)
            foreach (var act in acts)
                act(delta);
    }

    public override void _PhysicsProcess(double delta)
    {
        foreach (var (node, acts) in _physicsHooks)
            foreach (var act in acts)
                act(delta);
    }
}
```

✅ **How it works:**

* When any node enters the tree, this manager reflects over its methods.
* If it finds `[LunyReady]`, `[LunyProcess]`, or `[LunyPhysics]`, it hooks those automatically.
* The manager then calls those delegates every frame, without subclassing.

---

## 🧠 3. Example Usage in a Regular Godot Script

Users can now attach **plain** scripts to **any** node type — `Node2D`, `RigidBody3D`, `Area3D`, etc.
They don’t need to use custom base classes.

```csharp
// PlayerController.cs
using Godot;
using System;

public partial class PlayerController : Node3D
{
    private float _speed = 4.0f;

    [LunyReady]
    private void OnReady()
    {
        GD.Print("PlayerController ready!");
    }

    [LunyProcess]
    private void Tick(double delta)
    {
        var input = new Vector3(
            Input.GetActionStrength("move_right") - Input.GetActionStrength("move_left"),
            0,
            Input.GetActionStrength("move_back") - Input.GetActionStrength("move_forward")
        );
        Position += input.Normalized() * _speed * (float)delta;
    }
}
```

That’s it — no subclassing or special node type required. The manager automatically calls `OnReady()` and `Tick()` every frame.

---

## ⚙️ 4. Autoload Configuration

1. Save `LunyLifecycleManager.cs` in your Godot project.
2. In **Project Settings → Autoload**, add it as a singleton (check the box “Enable”).
3. Run your scene — the manager will start listening immediately.

---

## ⚡️ 5. Optional Extensions

You can easily extend this pattern with:

* `[LunyEnterTree]`, `[LunyExitTree]` attributes.
* `LunySignal` attributes for automatic signal connection.
* Tagging system: e.g. `[LunyTag("Enemy")]` for grouping.
* Parameter injection: pass `Node` reference or delta time automatically.

---

## 🧩 6. Why This Is Powerful

| Feature                      | Benefit                                                    |
| ---------------------------- | ---------------------------------------------------------- |
| Works with any built-in node | No need to subclass or modify C++                          |
| Attribute-driven             | Clear, declarative syntax                                  |
| Component-style              | Feels like Unity’s MonoBehaviour, but engine-agnostic      |
| Extensible                   | Add more attributes, integrate with LunyScratch logic tree |
| FOSS-friendly                | No editor hacks or engine forks                            |

---

Would you like me to extend this system so it supports **LunyScratch-specific components** — e.g. `@LunyEvent("when_key_pressed")` to automatically bind Scratch-like logic triggers at runtime?
That’s the natural next layer above this foundation.
