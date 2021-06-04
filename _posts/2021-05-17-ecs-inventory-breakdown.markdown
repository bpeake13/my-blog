---
layout: post
title:  "Global Game Jam ECS Inventory Breakdown"
date:   2021-05-23 16:10:06 -0700
categories: programming unity
---
# Overview
A few months ago was the 2021 Global Game Jam. I worked on a simple inventory management game [Bag Of Wonder][bag-of-wonder]. Before GGJ this year I had started playing with Unity's Entity Component System (ECS) packages, and went into the jam with the thought of using ECS to see how it fared under pressure. I didn't expect to have to make an inventory system and within the first few minutes I was already very worried, but I was persistent in trying. To my surprise it actually worked out OK. Better than OK really, what I thought was going to be a hard problem, actually ended up feeling way easier than any inventory system I've made in the past. So I was inspired to do a write up on my findings, and the design choices that led to these discoveries!

# A Brief Intro To ECS and DOP
Data Oriented Programming (DOP) is a methodology of programming, like Object Oriented Programming (OOP). In OOP, models are focused around creating objects with methodology and inheritance of traits. In DOP we don't focus around Objects, but rather data. Any construct is written as a piece of data, with no logic attached to it. Logic is instead written as separate pieces that take in and manipulate the data. Conceptually this can seem very similar at times, but where as in OOP logic is attached to its respective object, DOP does not have this restriction. This allows for logic to be written for processing many different pieces of data with ease.

ECS is an architecture primarily used in game development that uses DOP. In ECS, a game has Entities, Components, and Systems. Entities can be thought of as our game objects, like the player, enemies, coins, etc. Components in ECS work similarly to components in vanilla Unity; A user can create component types and attach them to entities. However these components do not have any logic, instead they only contain data, such as positions, velocity, score, etc. Users can then add systems to their game that query the world based on the types of components they need. 

Here is a short example of applying movement from velocity so you can see how it works. The system queries for all entities that match a pattern of data, and then applies a change to them.
```c#
struct Position : IComponentData {
  public Vector3 v;
}

struct Velocity : IComponentData {
  public Vector3 v;
}

class VelocitySystem : SystemBase {
  protected override void OnUpdate() {
    var deltaTime = Time.Delta;

    Entities.ForEach((ref Position pos, in Velocity velocity) => {
      pos.v += velocity.v * deltaTime;
    });
  }
}
```

# Writing Our Inventory
For `Bag Of Holding` we wanted to have a visual inventory system where users had to manage the inventory spatially, similar to `Diablo III`.

![Demo][demo-gif]

[bag-of-wonder]: https://globalgamejam.org/2021/games/bag-wonder-9

Throughout the game you get items in one slot periodically. The user can store these items in their inventory, and drag items into a box to use them. Different items will have different effects, and different cooldowns as to when you can use an item again.

The tricky part of the inventory was going to be structuring the data so we could keep track of slots being used. Weapons could take up variable numbers of slots based on shape and rotation. A users inventory can also grow in any way, so we can't just store simple arrays of what items we do and don't have.

We needed a simple database of items, and it had to be spatially based. Since ECS is already a database of objects, we had the idea "what if the world...is just the database". So we made a simple data setup, where items could be linked to multiple slots, and slots can be linked to only one item.

![Data Layout][data-layout]

If we want to add an item to the inventory, all we have to do is ensure the item is referencing the slot, and the slot is referencing  the item, that's it! Well, obviously that's not "ALL" that is happening, but that simple concept is really all that drives the inventory from a data point of view. Everything else about dragging, using, and so on is either a function, or a system that takes these in as parameters and reads and writes to it. For example the code that moves an item to the slots it’s equipped in, all it does is query for every item, and then moves each item to the average position of every slot the item is attached to. We even added a nice smooth movement for a nice visual effect.

```C#
Entities.WithEntityQueryOptions(EntityQueryOptions.FilterWriteGroup)
    .ForEach(
        (Entity e,
            DynamicBuffer<ItemAttachedBuffer> attachedBuffers,
            ref Translation itemTranslation,
            ref InternalItemMoveVelocity velocity,
            ref MoveableItem writeGroup,
            in ItemInstance _,
            in PhysicsCollider collider) => {
            if (attachedBuffers.Length == 0) {
                velocity.CurrentVelocity = Vector2.zero;
                return;
            }

            ...

            var aggregatePosition = float2.zero;
            for (int i = 0; i < attachedBuffers.Length; i++) {
                var slotEntity = attachedBuffers[i].Slot;
                if (slotEntity == Entity.Null) {
                    continue;
                }

                var slotTranslation = translationAccess[slotEntity];
                aggregatePosition += slotTranslation.Value.xy;
            }

            itemTranslation.Value.xy = Vector2.SmoothDamp(
                itemTranslation.Value.xy,
                (aggregatePosition / attachedBuffers.Length) - center.xy,
                ref velocity.CurrentVelocity,
                settings.SmoothTime,
                settings.MaxMoveSpeed,
                timeDeltaTime);
        })
    .Run();
```

Overall we only had a few core systems; item positioning, item dragging, and special slot systems. Each of these systems boiled down very similar to the system above, query either for the slots or spaces, then apply some transformation to the data based on the inventory state. Quite a few of the systems needed to share some rules. For example any system placing an item would need to follow the same rules of how and where you can place. To share these rules we wrote functions that took in the slots and items being attached as parameters. Now to share a rule, we only had to share the function call.

ECS even allowed for us to do some things we thought would be really hard, really easily, such as bridging between the inventory and the rest of the game systems, which were NOT ECS based. We thought passing data would be really hard. But we found we could bridge data between the two by attaching non-ECS objects as additional components. For example when we spawn an item,from  a scriptable object with stats and mesh from a list of items. We pass this to the inventory, where we create the correct Entity item, which contains the visuals for the item as well as stats and colliders, and attach it to the starting slot. Then when the item is used, it passes the scriptable object back to the game manager to process damage and gameplay. The amount of extra code to bridge the two is maybe 20 extra lines.

# Findings
The DOP approach ended up being a great way of handling our inventory. Under an OOP approach my findings for problems like this always boiled down to becoming complicated based on figuring out where logic should live, and making trade-offs based on where I chose to in the end. Make a choice and pay the price later. With this DOP approach however I never found myself getting backed into those corners. Extending the system was always super easy, because any piece of logic could have global access to everything, it didn't need to "live" on something and be restricted to influencing that one thing.

There are some definite down sides to this approach. For instance if we wanted to add a second player inventory, it would be a little difficult to do that with grace, and would definitely require code changes. With OOP this is something that is pretty much given for free. There is also the issue of code discoverability. In OOP if you wanna know how to use something, or where the code for something lives, you just find the class and look at it. In our case however the code is spread out over many functions and files as to how the inventory works, and there is no real unifying piece that ties them together beyond documentation. For me I didn't find these issues to out-weight the benefits of being able to solve a complex problem, pretty simply, without the usual growing pains or pit falls I've experienced before.

I very much encourage anyone who hasn't tried these programming styles to give them a try. Even if you don't adopt them fully, you may find a trick or two that will save you in the future!

If you're interested in learning more on these subjects, i recommend checking out these articles:
[Data Oriented vs Object Oriented Design][dops-article]
[Entities Components and Systems][ecs-article]

[demo-gif]: /my-blog/assets/BagOfWonder01.gif
[data-layout]: /my-blog/assets/ItemDataLayout.png
[dops-article]: https://medium.com/@jonathanmines/data-oriented-vs-object-oriented-design-50ef35a99056
[ecs-article]: https://medium.com/ingeniouslysimple/entities-components-and-systems-89c31464240d
