---
layout: post
title:  "Entities, Components, and Systems oh my!"
---

Traditional [object-orient programming](https://en.wikipedia.org/wiki/Object-oriented_programming) organizes code and the data that code manipulates into self-contained "objects". When done correctly, this can make it easy to understand how a complex system works since each class of object can be easily reasoned about in isolation. Code can be reused through [inheritance](https://en.wikipedia.org/wiki/Inheritance_(object-oriented_programming)) or [composition](https://en.wikipedia.org/wiki/Object_composition). This generally works well in many cases but isn't always the best way to model a system. 

Our game needs to simulate a solar system and also render that simulation. These are two very different bits of code, but they operate on the same base data (orbital positions over time). The rendering system builds on top of the simulation and has some additional data (models that are actually rendered). Currently, the code just mashes all that data into a single object and has code to handle both simulation and rendering interspersed. So, what other options do we have?

We can separate out the simulation and rendering logic fairly easily and just leave the data in our objects, but this still has the problem of needing to know all of the data that will be needed for a given object ahead of time so that we can include it in our classes. Currently, our data model looks something like this:

```typescript
interface SystemObject {
    name: string; // duplicated system.name
    radius: number; // duplicated system.radius
    orbit: Orbit; // duplicated orbital parameters and current position
    frame: THREE.Object3D;
    body: THREE.Object3D;
    label: HTMLCanvasElement | null;
    parent?: SystemObject;
    system: System; // the model object this is "extending"
}
```

We've got some duplicated data, two different bits of rendering info (Three.js objects to manage rendering and a canvas element to display the label), and some references to other related objects. What a mess! What we really need are extensible objects which can have different bits of data added and removed from them as needed based on the code that's processing those objects.

## Entity, Component, System
Enter [ECS](https://en.wikipedia.org/wiki/Entity_component_system), an architectural pattern which focuses on [composition over inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance) and replaces traditional objects with **Entities** that are essentially just containers for **Components** which contain the actual data. Since an entity can contain any number of components, those components can be fine-grained representations of a single aspect of the object (often only a single data field). Finally, **Systems** encapsulate the code which operates on these components.

This is much closer to how a schema-less database works where each entity can be though of as a row in a database table, components are the columns that hold data and systems are the code which queries rows with specific non-null columns and updates them. It is also particularly well suited for high performance applications with many objects that need to be processed in a tight loop, like games, as it can be efficiently implemented with a [data-oriented design](https://en.wikipedia.org/wiki/Data-oriented_design) to make optimal use of modern CPU instruction and data caches. This is the core idea behind [Unity DOTS](https://unity.com/dots).

After deciding to switch to an ECS system for this game, I looked into some options for existing JavaScript ECS systems, but didn't find anything I really liked, so I decided to setup my own simple system as a learning exercise and so that I can have full control over the details. I may end up replacing this with a [WASM](https://webassembly.org/)-based implementation written in [Rust](https://www.rust-lang.org/) later on (if/when additional performance is needed and as an excuse to finally learn Rust and WASM), but to get started and iterate quickly I'll be walking through a TypeScript implementation.

## Components
Let's start by looking at how we can decouple the rendering system from the underlying solar system model. Currently the rendering system builds the `SystemObject`s shown above for each body in the model during an init pass. Let's see if we can split this up into some more focused components:

```typescript
class Name {
    constructor(public name: string) { }
}

class Orbit {
    constructor(
        public a: number, // semi-major axis
        public e: number, // eccentricity
        public pomega: number, // logitude of periapsis = LAN + AOP
        public M0: number, // mean anomaly at epoch (J2000)
        public n: number, // mean angular motion (per day)
    ) { }
}

class LocalPosition {
    constructor(
        public x: number,
        public y: number,
    ) { }
}

class ThreeJsBody {
    constructor(
        public frame: THREE.Object3D,
        public body: THREE.Object3D,
    ) { }
}

class CanvasLabel {
    constructor(public label: HTMLCanvasElement) { }
}
```

The final code will have a few more components, but these are the ones that are are used throughout the rest of the post to demonstrate the design.

## Systems
Once we have the data broken out like this, we can start thinking about some of the systems we'll need to operate on this data. Firstly, we'll need a system to update the `LocalPosition` each frame based on the `Orbit` parameters and the current simulation time. Then we'll need another system to update the `frame` position in the `ThreeJsBody` component based on that `LocalPosition`. These system might look something like this:

```typescript
export interface System {
    readonly query: Query | Query[];
    run: (group: EntityGroup, world: World) => void;
}

class OrbitPositionUpdater implements System {
    readonly query = {
        all: [Orbit, LocalPosition],
    };

    run(group: EntityGroup) {
        group.with(Orbit).update(LocalPosition, (position, entity, orbit) => {
            // math here to update position from orbit + current simulation time
        });
    }
}

class ThreeJsFrameUpdater implements System {
    readonly query = {
        all: [LocalPosition, ThreeJsBody],
    };

    run(group: EntityGroup) {
        group.with(LocalPosition).update(ThreeJsBody, (three, entity, position) => {
            // copy position into three.frame.position
        });
    }
}
```

I've omitted the implementations since they're mostly irrelevant to building out the ECS system. The main point here is that we have a block of code that we want to execute on each entity that has a certain set of components. Our system then is just a `run()` method along with a `query` object to tell the system which components we care about.

The interesting bit here is that we're using the component class names as objects in both the query and the `EntityGroup` methods (`with()` and `update()`). Technically those are class constructors that are being passed around, the supporting types look something like this:

```typescript
type Constructor = new (...args: any[]) => any;

export class EntityGroup {
    readonly entities = new Set<Entity>();

    with<T extends Constructor[]>(...inputComponents: T): EntityGroupIterator<T> {
        return new EntityGroupIterator(this, ...inputComponents);
    }
}

class EntityGroupIterator<TInputs extends Constructor[]> {
    private readonly entities: Set<EntityImpl>;
    private readonly inputComponentClasses: TInputs;

    constructor(group: EntityGroup, ...inputComponentClasses: TInputs) {
        this.entities = group.entities as Set<EntityImpl>;
        this.inputComponentClasses = inputComponentClasses;
    }

    update<TUpdate extends Constructor>(componentClass: TUpdate, callback: UpdateComponentCallback<TUpdate, TInputs>) {
        for (const e of this.entities) {
            callback(e.get(componentClass)[0], e, ...e.get(...this.inputComponentClasses));
        }
    }
}
```

There's a bit more type-checking gymnastic going on behind the scenes, but the basic is that the system can request read-only access to whatever components it needs as input (`with(...components)`) and then `update()` a single component with a callback that receives all of these components (plus a reference to the underlying entity). Separate input and output components like this at the API level should open up some options for dependency tracking to enable parallel execution of systems in the future.

There's a bit of duplication here with the component names in the query and also in the processing loop. Unfortunately I haven't come up with a good way to eliminate this duplication just yet, but I've pretty happy with this API as-is and if I can tweak things a bit to remove that it will be even better!

Hopefully this has made sense so far, we can build systems that update components based on other components, but how do we handle initialization and tear down? The rendering system will need to add some components on mount and remove them on unmount, how might that work? With more systems of course!

```typescript
class CanvasLabelAdder implements System {
    readonly query = {
        all: [Name],
        none: [CanvasLabel],
    };

    run(group: EntityGroup) {
        group.with(Name).add((e, name) => {
            const element = new HTMLCanvasElement();
            // render the 'name' text onto this canvas here...
            return new CanvasLabel(element);
        });
    }
}

class CanvasLabelRemover implements System {
    readonly query = {
        all: [CanvasLabel],
    };

    run(group: EntityGroup) {
        group.remove(CanvasLabel);
    }

    afterAll(world: World) {
        world.remove(this);
    }
}
```

The first system would be added when the rendering system mounts, its query will match any entity with a `Name` component but no `CanvasLabel` component. Its job is to create and add that missing component to each entity. On unmount this system can be removed and the second system added instead. It matches any entity with a `CanvasLabel` which it simply removes. In addition, it uses a new `afterAll()` method to remove itself once it has completed. This is needed since the `run()` method can be called multiple times per update to allow some optimizations under the hood in how entities are stored, but we'll get to that later...

## Next Week
Next week I'll dive into how all of this is implemented under the hood. I'll do some thinking on the duplication issue I mentioned above to see if I can come up with a cleaver way to improve the API even more, and hopefully I'll have some checked in code with tests and everything to demonstrate everything working.
