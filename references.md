# References

These GitHub projects are useful comparison points for this repository.

They are **Unity-based**, so they are not direct drop-in equivalents to this Odin/Sokol blueprint, but they are good references for **how other game projects structure runtime loops, gameplay systems, feature modules, and rendering/gameplay boundaries**.

## ECS and data-oriented architecture

### 1. Entitas
- Repo: https://github.com/sschmid/Entitas
- Focus: Entity/component/system architecture for Unity and C#.
- Why it is relevant: good reference for how a gameplay codebase can move from a single megastruct toward grouped systems, filtered queries, and code-generated component access.
- Best use as a reference here: compare Blueprint's simple `Entity` megastruct and per-entity proc dispatch with a more formal ECS workflow.

### 2. EntityComponentSystemSamples
- Repo: https://github.com/Unity-Technologies/EntityComponentSystemSamples
- Focus: official Unity DOTS/ECS sample collection.
- Why it is relevant: shows how Unity structures system updates, data flows, baking, jobs, and rendering in a modern ECS stack.
- Best use as a reference here: compare this repo's single-frame update loop and manual batching against Unity's system-group driven world update model.

### 3. Latios-Framework
- Repo: https://github.com/Dreaming381/Latios-Framework
- Focus: advanced Unity ECS framework with custom bootstrap/modules for rendering, audio, physics, transforms, and VFX.
- Why it is relevant: strong example of a project that keeps a data-oriented core but grows into a larger feature toolkit without losing control over the runtime architecture.
- Best use as a reference here: useful when Blueprint starts needing dedicated subsystems for particles, transforms, spatial queries, audio, or higher-scale world simulation.

### 4. actors.unity
- Repo: https://github.com/PixeyeHQ/actors.unity
- Focus: hybrid ECS + classic Unity GameObject workflow with processors, layers, pooling, and reactive helpers.
- Why it is relevant: shows a middle ground between “just write gameplay directly” and “fully commit to formal ECS everywhere.”
- Best use as a reference here: good model if Blueprint grows toward layered gameplay processors, pooled world objects, or separate logic layers without giving up direct iteration speed.

## Modular and layered game architecture

### 5. ScriptableObject-Architecture
- Repo: https://github.com/DanielEverland/ScriptableObject-Architecture
- Focus: ScriptableObject-driven variables, events, runtime sets, and decoupled feature wiring.
- Why it is relevant: excellent example of using content assets plus event channels to reduce direct coupling between systems.
- Best use as a reference here: helpful if Blueprint grows enough global systems that `Game_State` becomes too dense and you want cleaner feature boundaries.

### 6. QFramework
- Repo: https://github.com/liangxiegame/QFramework
- Focus: layered Unity architecture built around Controller / System / Model / Utility / Command concepts.
- Why it is relevant: a strong contrast to Blueprint's intentionally flat structure; it shows what a more formal layered architecture looks like in practice.
- Best use as a reference here: useful when deciding whether a new feature should remain in `game.odin` or be promoted into a dedicated subsystem/module.

### 7. Unity-Programming-Patterns
- Repo: https://github.com/Habrador/Unity-Programming-Patterns
- Focus: Unity examples for game programming patterns like game loop, update method, component, observer, event queue, object pool, and state.
- Why it is relevant: not a framework, but a valuable architecture catalog for feature growth.
- Best use as a reference here: use it when adding systems such as pooling, event queues, finite state machines, UI flow, or command-based actions.

## Sample projects worth studying alongside the frameworks

### 8. lsss-wip
- Repo: https://github.com/Dreaming381/lsss-wip
- Focus: an open Unity DOTS project using Latios Framework.
- Why it is relevant: shows how a real gameplay project sits on top of a lower-level framework.
- Best use as a reference here: useful when thinking about how Blueprint might evolve from “template” into “actual shipped game structure” while keeping a coherent architecture.

### 9. Match-One
- Repo: https://github.com/sschmid/Match-One
- Focus: small Unity example project demonstrating Entitas in a full game context.
- Why it is relevant: easier to digest than a large framework repo and closer to the scope of a starter blueprint.
- Best use as a reference here: helpful when comparing small-project ergonomics between Blueprint's direct style and a more formal ECS setup.

## How to use these references

A practical way to use the list above is:

- study **Entitas** or **actors.unity** when you want richer gameplay-system organization,
- study **EntityComponentSystemSamples** or **Latios-Framework** when you want higher-scale data-oriented architecture,
- study **ScriptableObject-Architecture** when you want looser coupling between features,
- study **QFramework** when you want a strict layered architecture,
- study **Unity-Programming-Patterns** when you want pattern-level guidance before introducing a new subsystem,
- study **lsss-wip** or **Match-One** when you want to see how architectural ideas land in a concrete game.

## Closest matches to this repository's philosophy

If the goal is to find Unity projects that feel closest in spirit to this repository, start with:

1. **actors.unity** — because it balances direct iteration with structure.
2. **Entitas** — because it is a common next step when a simple gameplay architecture needs more explicit systems.
3. **Latios-Framework** — because it shows how far a custom game-first architecture can grow without turning into a generic engine.

Those three together give the best Unity-side comparison set for this Blueprint.
