# **Porting Mistreevous to C#: A High-Performance Behavior Tree Library for Modern .NET**

![MistreevousSharp - Behavior Tree Library for .NET](assets/mistreevous-thumb.png)

When I started working on dedicated servers for multiplayer games, I faced a practical challenge: I needed server entities to have sophisticated, structured, and consistent behaviors. In single-player games, this role typically belongs to the client. In modern multiplayer games, especially with authoritative servers, AI needs to run on the server — and this completely changes the performance and architecture context.

Behavior Trees were the natural solution, and the Mistreevous library, originally created in TypeScript by *nikkorn*, had the design I wanted to use: simple, modular, with a clear and extremely expressive DSL. But bringing it to the .NET ecosystem wouldn't just be translating code. It would mean reimagining its architecture for a more demanding environment, with a focus on performance, zero-allocation, and predictability.

This post documents that process.

---

# **Why Behavior Trees?**

Behavior Trees are widely used in games, robotics, simulations, and agent-based systems. Their strength comes from the combination of:

* hierarchical structure
* clear intent readability
* strong modularity
* ease of debugging
* predictable execution flow

They allow creating complex behaviors from simple building blocks.

In the context of multiplayer servers, this means:

* low cost per tick
* deterministic behavior
* separation between logic and state
* scalability with multiple entities

Mistreevous already provided this in TypeScript, but there was enormous room for a C# version that was **truly optimized for frame loops**.

---

# **Why Port Mistreevous?**

The original Mistreevous has:

* a fluid DSL
* well-defined nodes
* simple and easy-to-understand parsing
* JSON compatibility
* an accessible API

But it was created for a completely different ecosystem.

In C# — especially under .NET 9.0 and .NET Standard 2.1 — there are opportunities to:

* drastically reduce allocations
* optimize execution flow
* increase efficiency on servers
* use more suitable structures than JavaScript's `Array<T>`
* achieve better memory and GC predictability

If the original Mistreevous was elegant, the C# version needed to be elegant **and efficient at scale**.

---

# **It Wasn't Just a Port — It Was a Rewrite**

Even while respecting Mistreevous's semantics, the C# implementation needed to be restructured and optimized. Below is a summary of what had to change.

---

# **Zero-allocation: The Heart of the Port**

The main goal was to ensure that executing `Step()` — equivalent to processing one frame/tick of AI — **wouldn't generate garbage for the Garbage Collector**.

On servers, this is critical. Imagine dozens or hundreds of entities running AI 20–60 times per second. Any repeated allocation, even small, becomes a real problem.

The main techniques used were:

### **1. Complete Removal of LINQ in Hot Paths**

LINQ is expressive, but it creates enumerators, closures, temporary lists.

Snippets like:

```csharp
children.Where(x => condition(x))
```

were replaced with manual loops.

---

### **2. Intensive Use of Reusable Lists and Buffers**

Thread-safe, pre-allocated buffers, reused between ticks.

---

### **3. Manual Parsing Without Temporary Strings**

Instead of:

```csharp
var tokens = input.Split(' ');
```

character-by-character parsing was applied using Span/Index.

---

### **4. Avoiding Closures and Implicitly Generated Delegates**

Delegates generate captures that can allocate.

---

### **5. Traditional for Loop Iteration**

Avoids struct/class enumerators and all the machinery behind `foreach`.

---

### **6. Compact Nodes with Few Objects**

The TS version uses objects for everything.

The C# version uses:

* fixed arrays
* ref structs where applicable
* lightweight objects without superfluous fields

---

# **Full Compatibility with DSL and JSON**

One of the project's pillars was ensuring that any tree defined in the original Mistreevous:

```
root {
    sequence {
        action [CheckHealth]
        selector {
            action [Flee]
            action [Fight]
        }
    }
}
```

would work exactly the same in C#.

This involved:

* identical semantics for SUCCESS, FAILURE, and RUNNING
* guards with the same behavior
* same execution ordering
* compatibility with subtree references
* same evaluation flow for decorators

The goal was to allow developers to migrate or share behaviors between TS and C#.

---

# **Internal Project Architecture**

MistreevousSharp's organization was designed to reflect both the library's conceptual structure and facilitate maintenance, extensions, and internal optimizations. Below is an overview of the main folders and their responsibilities within the repository.

```
MistreevousSharp/
├── assets/                      → Logos, icons and images used in README
├── example/                     → Functional usage example (MyAgent and Program)
├── src/
│   └── Mistreevous/
│       ├── Agent.cs             → Represents the agent that executes the tree
│       ├── BehaviourTree.cs     → Core tree execution engine
│       ├── BehaviourTreeBuilder.cs
│       ├── BehaviourTreeDefinition.cs
│       ├── BehaviourTreeOptions.cs
│       ├── Lookup.cs
│       ├── MDSLDefinitionParser.cs
│       ├── Mistreevous.cs       → High-level API (entry point)
│       ├── NodeDetails.cs
│       ├── State.cs
│       ├── Utilities.cs
│
│       ├── Attributes/          → Attribute and callback system
│       │   ├── Attribute.cs
│       │   ├── Callbacks/
│       │   │   ├── Callback.cs
│       │   │   ├── Entry.cs
│       │   │   ├── Exit.cs
│       │   │   └── Step.cs
│       │   └── Guards/
│       │       ├── Guard.cs
│       │       ├── GuardPath.cs
│       │       ├── Until.cs
│       │       └── While.cs
│
│       ├── Nodes/               → Behavior tree node hierarchy
│       │   ├── Node.cs
│       │   ├── Composite.cs
│       │   ├── Decorator.cs
│       │   ├── Leaf.cs
│       │
│       │   ├── Composite/
│       │   │   ├── All.cs
│       │   │   ├── Lotto.cs
│       │   │   ├── Parallel.cs
│       │   │   ├── Race.cs
│       │   │   ├── Selector.cs
│       │   │   └── Sequence.cs
│       │
│       │   ├── Decorator/
│       │   │   ├── Fail.cs
│       │   │   ├── Flip.cs
│       │   │   ├── Repeat.cs
│       │   │   ├── Retry.cs
│       │   │   ├── Root.cs
│       │   │   └── Succeed.cs
│       │
│       │   └── Leaf/
│       │       ├── Action.cs
│       │       ├── Condition.cs
│       │       └── Wait.cs
│
│       ├── Optimizations/       → Internal utilities focused on zero-allocation
│       │   └── ZeroAllocationHelpers.cs
│
│       ├── Mistreevous.csproj   → Library project file
│       ├── bin/                 → Build output
│       └── obj/                 → Temporary compilation files
│
├── .github/workflows/           → NuGet publishing pipeline
│   └── publish.yml
├── README.md
├── CHANGELOG.md
└── tree.txt                     → Visual representation of structure (generated)
```

---

# **Technical Architecture Summary**

| Folder / File             | Responsibility                                                               |
| --------------------------- | ------------------------------------------------------------------------------ |
| **src/Mistreevous/**        | Contains all public API and library core                                      |
| **Nodes/**                  | Behavior tree implementation — composites, decorators and leaf nodes           |
| **Attributes/**             | Callback and guard system, compatible with original Mistreevous                |
| **Optimizations/**          | Critical code focused on zero-allocation and performance                      |
| **MDSLDefinitionParser.cs** | Parser for original Mistreevous DSL (MDSL)                                     |
| **BehaviourTree.cs**        | Execution engine, equivalent to the "engine"                                   |
| **Utilities.cs**            | Internal helper functions                                                     |
| **example/**                | Complete usage demonstration                                                  |
| **.github/workflows/**      | Automated build + NuGet publish pipeline                                      |

---

# **How This Architecture Relates to the TypeScript Version**

The organization above was inspired by the original Mistreevous, but adapted for C#:

* In the TypeScript version, nodes are simple objects; in C#, they became dedicated classes focused on performance.
* The DSL parsing was manually rewritten to maintain zero-allocation.
* The file hierarchy reflects Mistreevous's conceptual structure, facilitating migration and reading of the original code.
* The **Optimizations/** folder doesn't exist in the original project — it centralizes everything that's specific to .NET and C#.

---

# **Estimated Performance Gain**

Simple benchmarks (under low load) indicate:

| Operation           | Mistreevous TS | MistreevousSharp | Improvement            |
| ------------------ | -------------- | ---------------- | ------------------- |
| DSL Parsing        | ~0.8 ms        | ~0.35 ms         | ~2.3x               |
| Execution per tick | variable       | stable 0.00x ms  | substantial reduction |
| Allocations per tick | multiple      | zero             | eliminated          |

The real gain grows with the number of entities.

---

# **C# Usage Example**

```csharp
var definition = @"
root {
    sequence {
        action [CheckHealth]
        selector {
            action [Flee]
            action [Fight]
        }
    }
}";

var agent = new MyAgent();

var tree = new BehaviourTree(definition, agent);

while (gameRunning)
{
    tree.Step();
}
```

---

# **Practical Applications**

Beyond multiplayer servers, the library is suitable for:

* Unity (no GC spikes)
* Godot (C#)
* autonomous agents
* environment simulations
* embedded systems
* robots and drones with modular behavior

---

# **Next Steps**

I'm working on:

* advanced decorators (Timeout, Cooldown)
* parser improvements
* ScriptableObjects support in Unity
* visual editor (C#) to match the TS version

---

# **Conclusion**

Porting Mistreevous to C# was a challenging and rewarding project. More than converting code, it was necessary to reinterpret every detail for the .NET ecosystem, always with a focus on performance, predictability, and compatibility.

The result is a modern, lightweight library ready for high-demand applications — maintaining the simple and practical spirit of the original version.

**GitHub:** [https://github.com/guinhx/MistreevousSharp](https://github.com/guinhx/MistreevousSharp)
**NuGet:** [https://www.nuget.org/packages/MistreevousSharp](https://www.nuget.org/packages/MistreevousSharp)
**Original Mistreevous:** [https://github.com/nikkorn/mistreevous](https://github.com/nikkorn/mistreevous)
