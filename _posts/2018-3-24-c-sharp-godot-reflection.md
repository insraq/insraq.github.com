---
layout: post
title: "C# with Godot: Custom Attributes, Reflection and Extension Method"
---

It's been a long time [since I last wrote any serious C#](http://ruoyusun.com/2013/03/10/6-months-with-c-sharp.html) and my last experience was actually surprisingly pleasant (given that I am a long time Linux / Mac user). And I've been using [Godot](https://godotengine.org/) game engine for a long time for my games. When Godot 3.0 introduced C# support, I immediately decide to give it a whirl.

Godot, if you are not familiar with, comes with **GDScript**, which is a Python inspired scripting language for the engine. When writing a scene (think GameObject in Unity), it's common to write the following code:

```gdscript
var mylabel

func _ready():
    mylabel = get_node("MyLabel")
```

This basically gets hold of a node, which is commonly used. So GDScript provides a [syntactic sugar](http://docs.godotengine.org/en/3.0/getting_started/scripting/gdscript/gdscript_basics.html#onready-keyword) which basically does the same trick:

```gdscript
onready var mylabel = get_node("MyLabel")

func _ready():
    // No need to write anything here
```

However, in C# bindings, this is not provided, so you have to do something like this:

```c#
public class Char : Node
{
    private Label label1;
    private Label label2;
    private Label label3;
    public override void _Ready()
    {
        label1 = (Label) GetNode("MyLabel1");
        label2 = (Label) GetNode("MyLabel2");
        label3 = (Label) GetNode("MyLabel3");
    }
}
```

I've been thinking of ways to make this less cumbersome. From my very limited C# experience, I think it's possible to achieve this with a custom attribute, some reflections and extension method:

```c#
public class Char : Node
{
    [Node("MyLabel1")] private Label label1;
    [Node("MyLabel2")] private Label label2;
    [Node("MyLabel3")] private Label label3;
    public override void _Ready()
    {
        // Unfortunate this is line is unavoidable 
        // unless you can directly modify the base class, which we can't
        this.WireNodes();
    }
}
```

So let's first define our custom attribute.

```C#
using System;

[AttributeUsage(AttributeTargets.Field | AttributeTargets.Property)]
public class NodeAttribute : Attribute
{
    public string nodePath;

    public NodeAttribute(string np)
    {
        nodePath = np;
    }
}
```

After this, we have `[Node("nodePath")]`custom attribute. But this only defines the meta information: to make use of it, we need a bit more. **Extension methods** in C# allow us to add methods to an existing type without creating a new derived type or modifying the original type. To achieve this, we can define a static class with a special static method:

```c#
using Godot;
using System;
using System.Reflection;

public static class Extensions
{
    public static void WireNodes(this Node node)
    {

    }
}
```

To retrieve the information from attributes, we need a bit of reflection (*I know how you feel, but there's no way around it*):

```c#
public static class Extensions
{
    public static void WireNodes(this Node node)
    {
        // Search all fields in the class
        FieldInfo[] info = node
            .GetType()
            .GetFields(BindingFlags.DeclaredOnly | BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance);
        foreach (var f in info)
        {
            // Check whether the field has Node attribute
            NodeAttribute attr = (NodeAttribute)Attribute.GetCustomAttribute(f, typeof(NodeAttribute));
            if (attr != null)
            {
                // Get the node using NodePath and set the field value
                f.SetValue(node, node.GetNode(attr.nodePath));
            }
        }
    }
}
```

And there you go. As a static type language, I am pleasantly surprised at the extensibility of C#: it's relatively easy to extend the language and existing libraries without breaking static typing.