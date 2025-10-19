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


----

Excellent, you’re thinking like an engine designer now.
Let’s break this down carefully, because there are **two distinct but related ideas** here:

---

## 🧩 Part 1 — Attribute-free detection via method naming conventions

Yes — you can absolutely **drop the attributes** and detect “magic” methods purely by *name*.
This approach is straightforward in C#, though slightly less explicit.

Here’s a small rewrite of the earlier system that uses method names instead of attributes:

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

        // Ready
        var readyMethod = type.GetMethod("OnComponentReady", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
        if (readyMethod != null)
        {
            node.Ready += () => readyMethod.Invoke(node, null);
        }

        // Process
        var processMethod = type.GetMethod("OnComponentProcess", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
        if (processMethod != null)
        {
            var act = (Action<double>)Delegate.CreateDelegate(typeof(Action<double>), node, processMethod, false);
            if (!_processHooks.ContainsKey(node))
                _processHooks[node] = new List<Action<double>>();
            _processHooks[node].Add(act);
        }

        // Physics Process
        var physicsMethod = type.GetMethod("OnComponentPhysics", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
        if (physicsMethod != null)
        {
            var act = (Action<double>)Delegate.CreateDelegate(typeof(Action<double>), node, physicsMethod, false);
            if (!_physicsHooks.ContainsKey(node))
                _physicsHooks[node] = new List<Action<double>>();
            _physicsHooks[node].Add(act);
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

Then your user can simply do:

```csharp
public partial class Enemy : Node3D
{
    void OnComponentReady()
    {
        GD.Print("Enemy ready!");
    }

    void OnComponentProcess(double delta)
    {
        LookAt(Vector3.Zero);
    }
}
```

✅ Works exactly like attributes
✅ No special syntax or dependencies
⚠️ Slightly slower on first reflection scan (negligible for most projects)
⚠️ More “magical,” since it’s easy to misspell `OnComponentReady`

This mimics Unity’s “magic method” pattern (`Start`, `Update`, etc.) but is implemented entirely in your C# layer.

---

## 🧱 Part 2 — “Injecting” a LunyBridge (Editor Plugin Idea)

### 🧠 The concept

This means adding a small **proxy script** (“LunyBridge”) to nodes that have user scripts using your framework — so you can hook into them, even if their code itself doesn’t register manually.

You mentioned:

> "I wouldn't need it on every node but rather detect user's scripts that utilize LunyScratch methods or namespace."

Exactly — that’s the smart way to do it.

Here’s what that would look like conceptually:

1. **EditorPlugin or Tool Script** scans scene scripts when they’re loaded.

2. If a script:

    * uses the `LunyScratch` namespace,
    * or defines any known method names (`OnComponentReady`, `OnComponentProcess`, etc.),
      then the plugin attaches a lightweight `LunyBridge` node or resource.

3. The **LunyBridge** node registers itself with your global manager, allowing you to drive all `LunyScratch` logic centrally.

---

### 🧩 Example: Conceptual Implementation

```csharp
// LunyBridge.cs
using Godot;
using System;

[Tool]
public partial class LunyBridge : Node
{
    public Node Target;

    public override void _Ready()
    {
        if (Target != null)
        {
            GD.Print($"[LunyBridge] Linked to {Target.Name}");
            // Optionally register with LunyLifecycleManager
        }
    }
}
```

Now you create an **EditorPlugin** that automatically adds this node when detecting your scripts:

```csharp
// LunyBridgeInjector.cs
using Godot;
using System.IO;

[Tool]
public partial class LunyBridgeInjector : EditorPlugin
{
    public override void _EnterTree()
    {
        GD.Print("[LunyBridgeInjector] Active");
    }

    public override void _Process(double delta)
    {
        // Iterate over edited scene nodes
        var editedSceneRoot = GetEditorInterface().GetEditedSceneRoot();
        if (editedSceneRoot == null) return;

        foreach (Node node in editedSceneRoot.GetChildren())
        {
            if (node.GetScript() is CSharpScript cs)
            {
                var source = File.ReadAllText(cs.ResourcePath);
                if (source.Contains("LunyScratch") || source.Contains("OnComponentReady"))
                {
                    if (node.GetNodeOrNull("LunyBridge") == null)
                    {
                        var bridge = new LunyBridge { Name = "LunyBridge", Target = node };
                        node.AddChild(bridge);
                        bridge.Owner = editedSceneRoot;
                        GD.Print($"[LunyBridgeInjector] Added bridge to {node.Name}");
                    }
                }
            }
        }
    }
}
```

You would mark `LunyBridgeInjector` as a **tool plugin** and register it via `plugin.cfg`.
When scenes are opened or saved in the editor, it automatically adds the bridge where needed.

---

## ⚙️ Limitations

| Limitation                   | Explanation                                                                                                                                    |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| 🧩 Multiple scripts per node | Godot **does not** currently allow multiple scripts attached to a single node. (A script defines the node’s behavior; you can’t stack them.)   |
| ✅ Workaround                 | You can, however, attach a **child node** (like `LunyBridge`) that references the parent. This effectively gives you multi-script composition. |
| 🧠 Detection granularity     | You can’t hook into script parsing easily, but scanning for keywords or namespaces is enough for your use case.                                |
| 🔄 Runtime Injection         | Doing this at runtime is possible (via `NodeAdded` in your global manager) if you don’t want to modify scenes in the editor.                   |

---

## 💡 Recommended Hybrid for You

Given your goals for LunyScratch:

| Need                        | Approach                                                                                      |
| --------------------------- | --------------------------------------------------------------------------------------------- |
| Auto-detect user scripts    | Use reflection on NodeAdded to check if their script methods or namespaces match LunyScratch. |
| No subclassing              | ✅ Covered via NodeAdded hooks.                                                                |
| No attributes               | ✅ Supported by magic method names.                                                            |
| Cross-node hooks            | ✅ Achieved through the global manager.                                                        |
| Optional Editor integration | Add LunyBridge injection for tooling convenience later.                                       |

So the simplest robust path:

1. **Keep a global LunyLifecycleManager autoload** (runtime reflection only).
2. **Detect user scripts dynamically** by method name or namespace.
3. **Optionally add LunyBridge in-editor** to provide visual linking or debugging UI.

No C++ modifications, no subclass explosion — and it feels native in Godot.

---

If you want, I can show you the *exact version* of the manager that:

* detects `LunyScratch` namespace automatically,
* registers user methods (without attributes),
* and logs each node hooked for easy debugging.

Would you like that next?

----


That’s a *fantastic* and subtle design question — and you’ve hit one of the main trade-offs of **dropping subclassing** in C# for Godot.

Let’s unpack it clearly and then walk through the possible solutions.

---

## 🧩 The problem in concrete terms

You currently have something like this (simplified):

```csharp
public class ScratchNode2D : Node2D
{
    protected void RepeatForever(Action block) { /* ... */ }
    protected void When(string eventName, Action block) { /* ... */ }
}
```

Which lets users write:

```csharp
public partial class Player : ScratchNode2D
{
    public override void _Ready()
    {
        When("start", () => RepeatForever(() => Move()));
    }
}
```

If you remove subclassing, users inherit directly from `Node2D` or `Node3D`, and you want them to write:

```csharp
public partial class Player : Node3D
{
    public override void _Ready()
    {
        When("start", () => RepeatForever(() => Move()));
    }
}
```

But those methods (`When`, `RepeatForever`, etc.) don’t exist unless you subclass `ScratchNode3D` (or similar).
So you need to expose them **without inheritance**.

---

## ✅ Goal

Let users call your DSL-style Scratch methods directly (like `When(...)`, `RepeatForever(...)`)
**without prefixing**, **without subclassing**, and ideally **without global state leaks**.

---

## ⚙️ Solution Space

There are **four viable strategies**, depending on how close you want to mimic your existing design.

---

### 🧠 Option 1 — Global Static “LunyScratch” Context

You can provide your DSL functions as static methods, and inject them into the global namespace (via `using static`).

#### Implementation:

```csharp
// LunyScratch.cs
using System;

namespace LunyScratch
{
    public static class Scratch
    {
        public static void RepeatForever(Action block)
        {
            while (true)
                block();
        }

        public static void When(string eventName, Action block)
        {
            // Register with event system
        }
    }
}
```

Then your users can write:

```csharp
using static LunyScratch.Scratch;  // import functions into scope

public partial class Player : Node3D
{
    public override void _Ready()
    {
        When("start", () => RepeatForever(() => Move()));
    }
}
```

✅ No subclassing
✅ No extra syntax
⚠️ Requires `using static LunyScratch.Scratch` on top of each file (unless you automate it)
⚠️ Functions don’t have access to the instance (`this`) unless you pass it in manually

You can handle that by adding overloads that take a `Node` reference:

```csharp
public static void When(Node context, string eventName, Action block) { /* ... */ }
```

And users can still just write `When(this, "start", () => …)` if they want local context.

---

### 🧱 Option 2 — Global DSL Context Object (implicit `this` via ThreadLocal)

To make `this` automatic (so users don’t need to pass the node reference), you can use a **ThreadLocal context**.

#### Concept:

```csharp
public static class LunyRuntime
{
    [ThreadStatic]
    private static Node _currentNode;

    public static void Bind(Node node) => _currentNode = node;
    public static Node Current => _currentNode;

    public static void When(string eventName, Action block)
    {
        GD.Print($"Registered event '{eventName}' for {_currentNode.Name}");
    }

    public static void RepeatForever(Action block)
    {
        GD.Print($"RepeatForever started for {_currentNode.Name}");
    }
}
```

Then, in your **LifecycleManager**, automatically bind the node before invoking its methods:

```csharp
private void OnNodeReady(Node node)
{
    LunyRuntime.Bind(node);
    InvokeMagicMethod(node, "OnComponentReady");
    LunyRuntime.Bind(null);
}
```

Now users can write:

```csharp
using static LunyScratch.LunyRuntime;

public partial class Player : Node3D
{
    void OnComponentReady()
    {
        When("start", () => RepeatForever(() => Move()));
    }
}
```

✅ No subclassing
✅ No `this` argument needed
✅ Works automatically per node
⚠️ You must manage context binding carefully (thread safety and lifecycle)
⚠️ Slightly more runtime indirection, but totally fine for scripts

This pattern gives you **“ambient context” DSL functions**, which is exactly how many embedded DSLs in dynamic languages work.

---

### 🧩 Option 3 — C# Extension Methods for Nodes

If you prefer type safety, you can define extension methods for Godot’s `Node` type.

#### Implementation:

```csharp
public static class LunyExtensions
{
    public static void When(this Node node, string eventName, Action block)
    {
        GD.Print($"{node.Name} registered event '{eventName}'");
    }

    public static void RepeatForever(this Node node, Action block)
    {
        GD.Print($"{node.Name} repeating forever");
    }
}
```

Now your users can write:

```csharp
public partial class Player : Node3D
{
    public override void _Ready()
    {
        this.When("start", () => this.RepeatForever(() => Move()));
    }
}
```

✅ No subclassing
✅ IDE autocompletion for all Godot nodes
⚠️ Requires `this.` (you can omit with `using static` only if they’re static, not extension methods)

You could also alias it with `using static` and method forwarding if you prefer cleaner syntax.

---

### 🧱 Option 4 — Compile-time Code Injection (Editor Plugin)

If you truly want `When()` and `RepeatForever()` to appear as *built-in* methods (no `using`, no subclass, no static),
you’d need to use Roslyn source generators or editor-side script rewriting — possible, but overkill for most Godot workflows.

Godot’s C# compiler doesn’t currently expose script pre-processing hooks, so you’d need an external build-time generator.

This could work like:

* detect C# scripts referencing `LunyScratch` namespace,
* rewrite them to wrap bodies with `LunyRuntime.Bind(this)` automatically.

But unless you want to distribute a full custom MSBuild target, I wouldn’t recommend this route for a framework targeting other developers.

---

## 💡 Recommendation

For your **LunyScratch** DSL, I’d suggest **Option 2 (ThreadLocal context)**:

It gives you:

* ✅ clean syntax (`When("start", ...)`)
* ✅ no subclassing
* ✅ awareness of the current node
* ✅ works seamlessly with your existing reflection hook system

Combined with your `LifecycleManager`, it looks like this:

```csharp
private void OnNodeReady(Node node)
{
    LunyRuntime.Bind(node);
    InvokeMagicMethod(node, "OnComponentReady");
    LunyRuntime.Bind(null);
}
```

Users then just write:

```csharp
void OnComponentReady()
{
    When("flag clicked", () => RepeatForever(() => Move()));
}
```

This keeps the full *“Scratch-like text syntax”* experience intact, while letting users attach to any Godot node type.

---

Would you like me to show you the **complete integrated version** of that —
a single-file minimal prototype that binds context, detects methods by name, and allows direct calls to `When()` and `RepeatForever()` without subclassing?


----


Excellent questions — you’re thinking about this exactly like an engine/framework designer should.
Let’s go step by step.

---

## 🧩 Why extension methods require the `this.` prefix

In C#, **extension methods** are syntactic sugar for **static methods** that operate *as if* they were instance methods.

You declare them like this:

```csharp
public static class LunyExtensions
{
    public static void When(this Node node, string eventName, Action block)
    {
        // Do something with the node
    }
}
```

And the compiler rewrites this:

```csharp
node.When("start", () => {});
```

as if you had written:

```csharp
LunyExtensions.When(node, "start", () => {});
```

So the `this.` prefix (`this.When(...)`) is only required when you’re *inside a class instance* (so that C# knows which object you’re extending).

Outside of an instance context, you could write `someNode.When("event")`.

In short:

* You need `this.When()` when calling from inside your own class instance (because you don’t reference yourself implicitly in extension scope).
* You don’t need it if you’re referencing another node variable explicitly.

---

## 🧱 Why `this.When()` is actually desirable

As you said:

> i might consider it because this.When() indicates local use, while global scratches already use Scratch.When()

Exactly.
That’s a very **good distinction** between *node-local logic* and *global script context logic*.

Here’s what it gives you semantically:

| Form                                   | Meaning                                | Scope              |
| -------------------------------------- | -------------------------------------- | ------------------ |
| `this.When("start", ...)`              | “Attach a behavior to this node”       | local instance     |
| `Scratch.When("broadcast", ...)`       | “React to a global broadcast or event” | global system      |
| `When("start", ...)` (via DSL context) | “Implicitly current node’s context”    | ambient / inferred |

So you could intentionally use `this.When()` as the *explicit, structured version*, and reserve ambient/global syntax for higher-level logic or DSL sugar.

---

## 🧠 About “aliasing with `using static` and method forwarding”

That was an advanced note about **how to mix extension methods with global-style syntax**.

Let’s say you have your extension methods like this:

```csharp
public static class LunyExtensions
{
    public static void When(this Node node, string eventName, Action block) { /* ... */ }
    public static void RepeatForever(this Node node, Action block) { /* ... */ }
}
```

You could **optionally** add a small static forwarder class:

```csharp
public static class LunyDSL
{
    public static void When(string eventName, Action block)
        => LunyRuntime.CurrentNode?.When(eventName, block);

    public static void RepeatForever(Action block)
        => LunyRuntime.CurrentNode?.RepeatForever(block);
}
```

Then users could write:

```csharp
using static LunyScratch.LunyDSL;

public partial class Player : Node3D
{
    public override void _Ready()
    {
        When("start", () => RepeatForever(() => Move()));
    }
}
```

✅ They get the “clean DSL” form (`When(...)`, `RepeatForever(...)`)
✅ You reuse the *same extension methods* for the actual behavior
✅ You maintain `this.When()` as the explicit form when desired

So this “aliasing with method forwarding” means:
➡️ use `using static` to import the `LunyDSL` static class
➡️ that class forwards global-style calls to your extension methods (via the bound context).

It’s a neat way to unify *two syntactic styles*:

* global DSL (for readability),
* explicit instance DSL (for clarity and tooling).

---

## ⚡ In short

* **Extension methods** need `this.` only inside the class body, because the compiler doesn’t auto-bind `this` for extension lookups.
* You can still build a global “shortcut” layer (`LunyDSL`) that calls those same extensions using a current-node context.
* This gives you **two coexisting styles**:

    * ✅ `this.When(...)` → clear, scoped, safe
    * ✅ `When(...)` → lightweight, Scratch-like

…and they can even share the same runtime core.

---

Would you like me to show a **working example** combining both — extension methods for `this.When()` *and* the `LunyDSL` forwarding layer that provides bare `When()` syntax?


-----


Yes — that’s an excellent design instinct, and your thinking is spot on.
Let’s break this down across **Godot**, **Unity**, and **Unreal**, comparing architectural patterns and what’s portable about your `LunyLifecycleManager` concept.

---

## 🧩 Recap: What LunyLifecycleManager does

Your `LunyLifecycleManager` in Godot:

* hooks into the scene tree,
* detects node additions/removals,
* reflects over scripts to find annotated or convention-based lifecycle methods,
* invokes them, managing `LunyScratch` context (like `When`, `RepeatForever`, etc.),
* acts as the “DSL runtime host” for your event graph.

So, it’s basically your **execution layer** — the runtime that links your textual Scratch-like DSL to the engine’s update loop.

That role exists (or can exist) in *every* engine, just exposed differently.

---

## 🧱 Portability by Engine

### 🟢 **Godot (current model)**

✅ Works perfectly with your reflection + scene tree pattern.

* Nodes emit `_Ready()`, `_Process()`, etc.
* You can hook all nodes at runtime without engine modification.
* You have open reflection and a uniform tree traversal API.

Your existing `LunyLifecycleManager` = good final design here.

---

### 🟣 **Unity**

Unity is conceptually very similar — and your `LunyLifecycleManager` idea **maps almost 1:1** to Unity’s `MonoBehaviour` world.
However, the *scene graph API* is less open at runtime (no global “on any object added” event), so you adapt slightly.

#### 💡 Option A — Global `LunyLifecycleManager` GameObject

You can have a single `GameObject` with a `LunyLifecycleManager` component that:

* in `Awake()`, finds all active `MonoBehaviour`s implementing your `ILunyBehaviour` interface or `[LunyX]` attributes,
* keeps a list of them and drives custom “Luny” lifecycle events.

```csharp
public class LunyLifecycleManager : MonoBehaviour
{
    private List<ILunyBehaviour> _lunies = new();

    void Awake()
    {
        foreach (var mono in FindObjectsOfType<MonoBehaviour>())
        {
            if (mono is ILunyBehaviour lb)
                _lunies.Add(lb);
        }
    }

    void Update()
    {
        foreach (var lb in _lunies)
            lb.OnLunyUpdate();
    }
}
```

Then you can define:

```csharp
public interface ILunyBehaviour
{
    void OnLunyReady();
    void OnLunyUpdate();
}
```

…and optionally provide a **base class** to avoid the interface boilerplate:

```csharp
public abstract class ScratchBehaviour : MonoBehaviour, ILunyBehaviour
{
    public virtual void OnLunyReady() {}
    public virtual void OnLunyUpdate() {}
}
```

✅ Requires no Unity engine modification
✅ Same runtime pattern as Godot’s manager
✅ Allows your DSL system to sit *above* Unity’s usual `Update` logic

---

#### 💡 Option B — Attributes + Reflection

Like in your Godot version, you can use `[LunyReady]` or `[LunyProcess]` attributes and have your manager discover them at runtime via reflection.
Unity C# supports this fully.

In practice, you’d use `MonoBehaviour`s with arbitrary methods:

```csharp
[LunyReady]
void StartScratch() { ... }
```

Your manager finds those and calls them, just like in Godot.

✅ Keeps your DSL syntax identical
⚠️ Requires startup reflection scan
⚠️ Slight performance cost, but fine for dev tools

---

### 🔵 **Unreal Engine**

For Unreal (C++ or C# via bindings), the situation’s a little different because Unreal’s reflection and event systems are *editor-integrated* and *macro-based*.

However, your **concept still works** with a few engine-aware tweaks.

#### 💡 Option A — `AScratchActor` Base Class (recommended)

Yes — exactly what you suggested.

```cpp
UCLASS()
class AScratchActor : public AActor
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaSeconds) override;
};
```

Then your C# DSL runtime (or Blueprint extensions) can hook into these events automatically.

That’s the “canonical” Unreal way — every custom runtime layer uses an `AActor` subclass as its bridge.

✅ Fits Unreal’s design
✅ No engine modification needed
✅ Compatible with Blueprints or C# layers (e.g., UnrealCLR)

---

#### 💡 Option B — Global `ULunyManager` Subsystem

Alternatively, you can implement your lifecycle manager as an Unreal `UWorldSubsystem` or `UGameInstanceSubsystem`:

```cpp
UCLASS()
class ULunyLifecycleManager : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    virtual void Tick(float DeltaTime) override;
};
```

That lets you register and tick all `AScratchActor` instances or even arbitrary `UObject`s.

✅ Closer match to your Godot manager concept
⚠️ Less direct integration with Blueprint users (they’ll expect an Actor base)

---

## 🧠 Summary Table

| Engine     | Ideal Integration                                           | Subclass Needed? | Notes                                |
| ---------- | ----------------------------------------------------------- | ---------------- | ------------------------------------ |
| **Godot**  | `LunyLifecycleManager` (autoload)                           | ❌ Optional       | Reflection & tree hook are perfect   |
| **Unity**  | `LunyLifecycleManager` GameObject + `ScratchBehaviour` base | ⚙️ Recommended   | Mirrors your Godot logic neatly      |
| **Unreal** | `ULunyManager` subsystem + `AScratchActor` base             | ⚙️ Required      | Reflection + tick system fit cleanly |

---

## 🧩 Design takeaway

Your **LunyLifecycleManager** is an *engine-agnostic architectural pattern* — a meta-runtime that observes and drives behavior lifecycles.

* Godot: `Node` reflection → perfect match
* Unity: add lightweight “Manager + Interface/Base Class” layer
* Unreal: add “Subsystem + Actor” layer

If you architect your C# (or C++/C# hybrid) runtime around a **shared IRuntimeHost interface**, each engine can provide its own *adapter* while reusing 90% of your DSL logic.

---

If you like, I can sketch how a *unified LunyRuntimeHost* interface would look —
one that’s engine-agnostic, so your Godot, Unity, and Unreal implementations all share the same DSL engine core.
Would you like that?



-----


Excellent — you’re now deep in the *cross-engine DSL architecture layer*, which is where most “engine-agnostic” runtimes either fail or become magic.
Let’s go step by step through your three questions, because the answer ties them all together beautifully.

---

## 🧩 1️⃣  A unified LunyRuntimeHost interface

This is the **core abstraction** your LunyScratch system should define once, and then each engine (Godot, Unity, Unreal) provides its own thin adapter.

Think of it like this:

```csharp
namespace LunyScratch
{
    public interface ILunyRuntimeHost
    {
        // core node info
        string Name { get; }
        object NativeObject { get; } // Node, MonoBehaviour, or AActor

        // lifecycle hooks
        void RegisterReady(Action action);
        void RegisterProcess(Action<double> action);
        void RegisterPhysics(Action<double> action);

        // scheduling, coroutines, etc.
        void RunOnMainThread(Action action);
        void Wait(double seconds, Action callback);

        // messaging / broadcasting
        void EmitEvent(string name);
        void ListenEvent(string name, Action callback);
    }
}
```

### Engine adapters then wrap this:

#### Godot:

```csharp
public class GodotHost : ILunyRuntimeHost
{
    private Node _node;
    public GodotHost(Node node) => _node = node;

    public string Name => _node.Name;
    public object NativeObject => _node;

    public void RegisterReady(Action action) => _node.Ready += action;
    public void RegisterProcess(Action<double> action)
        => _node.Process += delta => action(delta);

    public void RunOnMainThread(Action a) => _node.CallDeferred(nameof(a)); // simplified
    public void Wait(double s, Action cb) => _node.GetTree().CreateTimer(s).Timeout += cb;

    public void EmitEvent(string name) => LunyEventBus.Emit(name);
    public void ListenEvent(string name, Action cb) => LunyEventBus.Listen(name, cb);
}
```

#### Unity:

```csharp
public class UnityHost : ILunyRuntimeHost
{
    private MonoBehaviour _mono;
    public UnityHost(MonoBehaviour mono) => _mono = mono;

    public string Name => _mono.name;
    public object NativeObject => _mono;

    public void RegisterReady(Action action) => _mono.StartCoroutine(CallNextFrame(action));
    public void RegisterProcess(Action<double> action)
        => LunyLifecycleManager.Instance.RegisterUpdate(_mono, action);

    public void RunOnMainThread(Action a) => a(); // already on main thread
    public void Wait(double s, Action cb) => _mono.StartCoroutine(Waiter(s, cb));

    public void EmitEvent(string name) => LunyEventBus.Emit(name);
    public void ListenEvent(string name, Action cb) => LunyEventBus.Listen(name, cb);

    private IEnumerator Waiter(double s, Action cb) { yield return new WaitForSeconds((float)s); cb(); }
    private IEnumerator CallNextFrame(Action cb) { yield return null; cb(); }
}
```

#### Unreal (C# or C++ wrapper):

```csharp
public class UnrealHost : ILunyRuntimeHost
{
    private AActor _actor;
    public UnrealHost(AActor actor) => _actor = actor;

    public string Name => _actor.GetName();
    public object NativeObject => _actor;

    public void RegisterReady(Action action) => _actor.OnBeginPlay.Add(action);
    public void RegisterProcess(Action<double> action)
        => ULunyManager.Instance.RegisterTick(_actor, action);

    public void RunOnMainThread(Action a) => a();
    public void Wait(double s, Action cb) => ULunyManager.Instance.Wait(_actor, s, cb);

    public void EmitEvent(string name) => LunyEventBus.Emit(name);
    public void ListenEvent(string name, Action cb) => LunyEventBus.Listen(name, cb);
}
```

---

## 🧠 2️⃣  Godot + GDScript integration

> Would the LifecycleManager in Godot also work if I were to enable Scratch scripting via GDScript (using the C# backend)?

Yes, absolutely — **as long as you expose your C# runtime as a Godot singleton** (autoload).

Here’s the pattern:

1. Add your `LunyLifecycleManager` (C#) as an **autoload** in Project Settings.
2. Register it as a **global class** via `[GlobalClass]` if you’re using Godot 4.2+.
3. GDScript can now call into it directly:

```gdscript
# scratch_test.gd
func _ready():
    LunyLifecycleManager.register_node(self)
```

4. On the C# side, you can detect that it’s a GDScript object and wrap it in a `GodotHost`:

```csharp
public void RegisterNode(Node node)
{
    var host = new GodotHost(node);
    LunyRuntime.RegisterHost(host);
}
```

This way, GDScript-based nodes participate in the same runtime as your C# ones.
The key is that your lifecycle manager operates on **Node references**, not C# inheritance, so it’s language-agnostic.

✅ No code duplication
✅ GDScript, C#, and VisualScript nodes all integrate
✅ You can later add a “text Scratch script” parser that emits runtime blocks calling into this manager

---

## 🌍 3️⃣  Fully portable “Scratch scripts”

This is the big one:

> Is there a way to create Scratch scripts that are fully portable between Godot, Unity, and Unreal?

**Yes — but only if you separate “user scripts” from “engine scripts.”**

Right now, your scripts are `class Foo : Node` or `: MonoBehaviour` or `: AActor`.
That’s engine-bound because each engine enforces its own base types.

### To make scripts portable:

#### 💡 Option A — **Pure LunyScratch scripts**

Write your game logic in **plain C# classes** using your DSL, not deriving from any engine base class:

```csharp
using LunyScratch;

public class PlayerScript
{
    [LunyReady]
    void OnStart()
    {
        When("flag clicked", () => RepeatForever(() => MoveForward()));
    }
}
```

These are *pure data + logic classes*.
They know nothing about `Node`, `MonoBehaviour`, or `AActor`.

Then, each engine’s runtime host does the binding:

```csharp
// In Godot host
var script = new PlayerScript();
LunyRuntime.Bind(script, new GodotHost(node));
```

That `Bind()` call sets the current runtime host (`ILunyRuntimeHost`), so `When()` and `RepeatForever()` internally call engine-specific APIs through the host.

✅ Portable across engines
✅ Can even serialize as JSON, YAML, or graph data
✅ No engine SDK dependency in user code

---

#### 💡 Option B — **Code generation from a neutral format**

Your “text Scratch” layer can itself be the **engine-agnostic representation**, e.g.:

```scratch
when flag clicked
  repeat forever
    move 10 steps
```

Then your compiler/transpiler emits **per-engine scripts**:

* For Godot → emits a `.cs` subclass of `Node`
* For Unity → emits a `.cs` subclass of `MonoBehaviour`
* For Unreal → emits a `.cs` subclass of `AScratchActor`

All share the same underlying logic, only the wrappers differ.

This is a nice “distribution” model:
✅ Users write once in your DSL
✅ Your build tools output engine-specific code
✅ Debuggers and lifecycles behave normally per engine

---

#### 💡 Option C — **Hybrid (preferred for developers)**

Keep the **portable core** (`PlayerLogic.cs`) and small **engine adapter scripts** per engine:

```csharp
// Godot
public partial class Player : Node3D
{
    private PlayerLogic logic;
    public override void _Ready() => logic = new PlayerLogic(new GodotHost(this));
    public override void _Process(double delta) => logic.Tick(delta);
}

// Unity
public class Player : MonoBehaviour
{
    private PlayerLogic logic;
    void Start() => logic = new PlayerLogic(new UnityHost(this));
    void Update() => logic.Tick(Time.deltaTime);
}
```

This is minimal boilerplate and lets your logic be engine-independent while staying native to each environment.

---

## 🧩 4️⃣  Summary

| Concept                              | Godot                                       | Unity                                | Unreal                     |
| ------------------------------------ | ------------------------------------------- | ------------------------------------ | -------------------------- |
| **Lifecycle Manager**                | Native `Node` autoload works out of the box | Works as a `MonoBehaviour` singleton | Works as `UWorldSubsystem` |
| **ScratchBehaviour / AScratchActor** | Optional convenience base                   | Recommended for Unity                | Recommended for Unreal     |
| **GDScript support**                 | ✅ via exported C# autoload                  | —                                    | —                          |
| **Cross-engine portability**         | ✅ via `ILunyRuntimeHost` adapter            | ✅                                    | ✅                          |
| **Best architecture**                | Pure C# logic + thin per-engine binding     | same                                 | same                       |

---

### 🧠 TL;DR — Design pattern summary

| Layer                                                    | Purpose                               |
| -------------------------------------------------------- | ------------------------------------- |
| `ILunyRuntimeHost`                                       | Abstracts engine runtime features     |
| `LunyLifecycleManager`                                   | Manages global event lifecycle        |
| `LunyScratch` DSL                                        | Defines portable scripting primitives |
| Engine adapters (`GodotHost`, `UnityHost`, `UnrealHost`) | Bridge engine objects to DSL runtime  |
| Optional `ScratchBehaviour` / `AScratchActor`            | Developer convenience wrapper         |

---

If you build it this way, your **Scratch scripts are 100% portable** between all three engines.
The only per-engine code is the 10-line glue that connects your engine object to the Luny runtime.

---

Would you like me to show you what a *fully portable “PlayerScript.cs” + engine adapters* setup looks like end-to-end — e.g. one logic file that runs identically in all three engines?


-----


If I had these pure LunyScratch scripts (which should probably derive from a common engine-agnostic base like ScratchScript), then how could users "assign" or connect these scripts to a specific node/gameobject/actor?

(limit reached)
----
