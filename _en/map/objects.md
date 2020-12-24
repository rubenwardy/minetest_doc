---
title: Objects, Players, and Entities
layout: default
root: ../..
idx: 3.4
description: Using an ObjectRef
degrad:
    level: warning
    title: Degrees and Radians
    message: Attachment rotation is set in degrees, whereas object rotation is in radians.
            Make sure to convert to the correct angle measurement.
---

## Introduction <!-- omit in toc -->

In this chapter, you will learn how to manipulate objects and how to define your
own.

- [What are Objects, Players, and Entities?](#what-are-objects-players-and-entities)
- [Position and Velocity](#position-and-velocity)
- [Object Properties](#object-properties)
- [Entities](#entities)
- [Attachments](#attachments)
- [Your Turn](#your-turn)

## What are Objects, Players, and Entities?

Players and Entities are both types of Objects. An Object is something that can move
independently of the node grid and has properties such as velocity and scale.
Objects aren't items, and they have their own separate registration system.

There are a few differences between Players and Entities.
The biggest one is that Players are player-controlled, whereas Entities are mod-controlled.
This means that the velocity of a player cannot be set by mods - players are client-side,
and entities are server-side.
Another difference is that Players will cause map blocks to be loaded, whereas Entities
will just be saved and become inactive.

This distinction is muddied by the fact that Entities are controlled using a table
which is referred to as a Lua entity, as discussed later.

## Position and Velocity

`get_pos` and `set_pos` exist to allow you to get and set an entity's position.

```lua
local object = minetest.get_player_by_name("bob")
local pos    = object:get_pos()
object:set_pos({ x = pos.x, y = pos.y + 1, z = pos.z })
```

`set_pos` immediately sets the position, with no animation. If you'd like to
smoothly animate an object to the new position, you should use `move_to`.
This, unfortunately, only works for entities.

```lua
object:move_to({ x = pos.x, y = pos.y + 1, z = pos.z })
```

An important thing to think about when dealing with entities is network latency.
In an ideal world, messages about entity movements would arrive immediately,
in the correct order, and with a similar interval as to how you sent them.
However, unless you're in singleplayer, this isn't an ideal world.
Messages will take a while to arrive. Position messages may arrive out of order,
resulting in some `set_pos` calls being skipped as there's no point going to
a position older than the current known position.
Moves may not be similarly spaced, which makes it difficult to use them for animation.
All this results in the client seeing different things to the server, which is something
you need to be aware of.

## Object Properties

Object properties are used to tell the client how to render and deal with an
object. It's not possible to define custom properties, because the properties are
for the engine to use, by definition.

Unlike nodes, objects have a dynamic rather than set appearance.
You can change how an object looks, among other things, at any time by updating
its properties.

```lua
object:set_properties({
    visual      = "mesh",
    mesh        = "character.b3d",
    textures    = {"character_texture.png"},
    visual_size = {x=1, y=1},
})
```

The updated properties will be sent to all players in range.
This is very useful to get a large amount of variety very cheaply, such as having
different skins per-player.

As shown in the next section, entities can have initial properties
provided in their definition.
The default Player properties are defined in the engine, however, so you'll
need to use `set_properties()` in `on_joinplayer` to set the properties for newly
joined players.

## Entities

An Entity has a definition table that resembles an item definition table.
This table can contain callback methods, initial object properties, and custom
members.

```lua
local MyEntity = {
    initial_properties = {
        hp_max = 1,
        physical = true,
        collide_with_objects = false,
        collisionbox = {-0.3, -0.3, -0.3, 0.3, 0.3, 0.3},
        visual = "wielditem",
        visual_size = {x = 0.4, y = 0.4},
        textures = {""},
        spritediv = {x = 1, y = 1},
        initial_sprite_basepos = {x = 0, y = 0},
    },

    message = "Default message",
}

function MyEntity:set_message(msg)
    self.message = msg
end
```

Entity definitions differ in one very important way from Item definitions.
When an entity is emerged (ie: loaded or created), a new table is created for
that entity that *inherits* from the definition table.

<!--
This inheritance is done using a Metatables are an important Lua feature that you will need
to be aware of, as it is an essential part of the Lua language.
In layman's terms, a metatable allows you to control how the table behaves when
using certain Lua syntax. The most common use of metatables is the ability to use
another table as a prototype, defaulting to the other table's properties and methods when
they do not exist in the current table.
Say you want to access `a.x`. If the table `a` has that member, then it will be
returned as normal. However, if the table doesn't have that member and the
metatable lists a table `b` as a prototype, then table `b` will be checked to see
if it has that member.
-->

Both an ObjectRef and an entity table provide ways to get the counterpart:

```lua
local entity = object:get_luaentity()
local object = entity.object
print("entity is at " .. minetest.pos_to_string(object:get_pos()))
```

There are a number of available callbacks for use with entities.
A complete list can be found in [lua_api.txt](https://minetest.gitlab.io/minetest/minetest-namespace-reference/#registered-definition-tables).

```lua
function MyEntity:on_step(dtime)
    local pos      = self.object:get_pos()
    local pos_down = vector.subtract(pos, vector.new(0, 1, 0))

    local delta
    if minetest.get_node(pos_down).name == "air" then
        delta = vector.new(0, -1, 0)
    elseif minetest.get_node(pos).name == "air" then
        delta = vector.new(0, 0, 1)
    else
        delta = vector.new(0, 1, 0)
    end

    delta = vector.multiply(delta, dtime)

    self.object:move_to(vector.add(pos, delta))
end

function MyEntity:on_punch(hitter)
    minetest.chat_send_player(hitter:get_player_name(), self.message)
end
```

Now, if you were to spawn and use this entity, you'd notice that the message
would be forgotten when the entity becomes inactive then active again.
This is because the message isn't saved.
Rather than saving everything in the entity table, Minetest gives you control over
how to save things.
Staticdata is a string which contains all the custom information that
needs to stored.

```lua
function MyEntity:get_staticdata()
    return minetest.write_json({
        message = self.message,
    })
end

function MyEntity:on_activate(staticdata, dtime_s)
    if staticdata ~= "" and staticdata ~= nil then
        local data = minetest.parse_json(staticdata) or {}
        self:set_message(data.message)
    end
end
```

Minetest may call `get_staticdata()` as many times as it wants and at any time.
This is because Minetest doesn't wait for a MapBlock to become inactive to save
it, as this would result in data loss. MapBlocks are saved roughly every 18
seconds, so you should notice a similar interval for `get_staticdata()` being called.

`on_activate()`, on the other hand, will only be called when an entity becomes
active either from the MapBlock becoming active or from the entity spawning.
This means that staticdata could be empty.

Finally, you need to register the type table using the aptly named `register_entity`.

```lua
minetest.register_entity("mymod:entity", MyEntity)
```

The entity can be spawned by a mod like so:

```lua
local pos = { x = 1, y = 2, z = 3 }
local obj = minetest.add_entity(pos, "mymod:entity", nil)
```

The third parameter is the initial staticdata.
To set the message, you can use the entity table method:

```lua
obj:get_luaentity():set_message("hello!")
```

Players with the *give* [privilege](../players/privileges.html) can
use a [chat command](../players/chat.html) to spawn entities:

    /spawnentity mymod:entity

## Attachments

Attached objects will move when the parent - the object they are attached to -
is moved. An attached object is said to be a child of the parent.
An object can have an unlimited number of children, but at most one parent.

```lua
child:set_attach(parent, bone, position, rotation)
```

An Object's `get_pos()` will always return the global position of the object, no
matter whether it is attached or not.
`set_attach` takes a relative position, but not as you'd expect.
The attachment position is relative to the parent's origin as scaled up by 10 times.
So, `0,5,0` would be half a node above the parent's origin.

{% include notice.html notice=page.degrad %}

For 3D models with animations, the bone argument is used to attach the entity
to a bone.
3D animations are based on skeletons - a network of bones in the model where
each bone can be given a position and rotation to change the model, for example,
to move the arm.
Attaching to a bone is useful if you want to make a character hold something:

```lua
obj:set_attach(player,
    "Arm_Right",           -- default bone
    {x=0.2, y=6.5, z=3},   -- default position
    {x=-100, y=225, z=90}) -- default rotation
```

## Your Turn

* Make a windmill by combining nodes and an entity.
* Make a mob of your choice (using just the entity API, and without using any other mods).
