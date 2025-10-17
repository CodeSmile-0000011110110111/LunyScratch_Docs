# Engine Notes

Observations, complaints, niceties.


## Godot 4.5


### Pros

- The little "reset to default value" circles in Inspector are welcome!
- C# debugging: debugger attaches much faster than with Unity. Debug button will automatically play the game too!
- Rigidbody: can specify initial and constant forces

### Cons

- UX
- Overall Godot requires a lot more extra steps and clicks to create things than in Unity or Unreal. This has repercussions for learning Godot as you have to memorize a lot more individual steps and the chances of doing something incorrect increases.
- For example, to create a collidable floor plane in Unity, you simply create a "3D/Plane" object and you are set. In Godot, you have to create a MeshInstance3D node, add a StaticBody3D child to it, and add a CollisionShape3D child to the body. Then edit the Shape3D to select the BoxShape3D, then edit its position and scale so the collider matches the plane mesh. Oh and yes, you need to add a camera and directional light if you want to see anything! And enable shadows on the directional light. And you'll likely need a WorldEnvironment. And then add an environment. And then set its mode to render a sky. ..........
- Unhelpful scene save failure message box: "Couldn't save scene. Likely dependencies (instances or inheritance) couldn't be satisfied."
- Moving tscn files may cause Godot not to find them anymore!!! 
- Too many modal dialogs (settings specifically). Particularly annoying for settings that require interaction with the non-modal UI to see its effect, which means open/close settings a lot after every change.
- Many settings changes require restarting the editor

- The Node System
- Well, it's an OOP hierarchy and we all know what's wrong with creating a deeply nested hierarchical OOP hierarchy, especially when many of the instance do similar but not identical things. As an arbitrary example, we have Shape and Polygon nodes, but what if we needed a PolygonShape node? Since we cannot combine the two as components, it means the number of nodes will grow much faster than in a component system, and it is less flexible. Likewise, there will be more Inspector settings than necessary. For instance a WorldEnvironment also has Physics Settings (Interpolation), which absolutely makes no sense. 
- It complicates type checking. For instance, if you want to set the Visible state of a Node parameter, you have to test whether it's a Node2D, Node3D, or any other type of Node subclass that has the Visible state. And non-intuitively, while Node3D inherits directly from Node, the Node2D actually inherits from CanvasItem. 
- In C#, if the script class (ie Node3D) does not match the node type (ie Rigidbody3D) then this fails: if (n is Rigidbody3D) >> false!!
- Which in turn means, you may need to create more "base" classes than you expected, especially if you intend to design a flexible system. 
- Confusingly named nodes. For instance, what's the difference between a Polygon and a Shape? So I have to make a decision between CollisionShape3D and CollisionPolygon3D and the description isn't helping either. It's non-intuitive, and not beginner-friendly. See also: https://www.reddit.com/r/godot/comments/x3thp8/why_are_there_separate_collisionpolygon2d_and/
- Seemingly arbitrary hierarchy requirements. For instance a Rigidbody must have a child collision node. You might do it the other way, there's no telling which is correct until you try. And then you want to add some visuals to it. Do you add the rigidbody as child to the sprite/mesh? No dummy, the sprite/mesh "follows" the rigidbody, which is counter-intuitive because we're used to thinking that the visualization is the object, and then we add functional stuff to it. 

- Scenes / Prefabs
- One would think that creation a MeshInstance3D prefab scene is most common, yet this requires selecting a custom node. Otherwise using Scene3D will add another level of indirection which may not be desirable.

- FileSystem view
- The FileSystem view shows the root of the project. This is troublesome for source control unless the Godot project itself is a subfolder in the repository root. Problem is that README.md and similar non-project files/folders will show up in Godot's project view where they do not belong. Unity and Unreal projects both use subfolders to which their project views are rooted. 
- For whatever reason, I keep creating (saving) scenes whose .tscn ends up being in the wrong folder (typically root). It feels like more often than not, merely selecting a folder in FileSystem has no effect, you have to double-click it??

- Inspector
- The more complex a node, the more settings there are. These can quickly become overhwelming. Nesting makes this even worse
- Transform settings are the most important settings for any note. Yet they are always below more specialized properties, pushing the Transform more and more to the bottom. Which also means Transform is only ever in the same relative place for nodes of the same type. 
- There is no visual highlighting of individual settings groups. This makes it hard to distinguish between groups, for example visual settings vs physics settings vs metadata. 
- There is also no filtering of Inspector items, other than by name.
- Inspector settings are per node, not per node type! This means the collapsed / expanded state of Inspector items changes as you select different nodes even of the same type. Confusing, as it unexpectedly shifts locations of Inspector items. And in multi-selection, even if all nodes have a group expanded, the multiselection Inspector will have that group collapsed!
- Inspector value changes only take effect when using slider. Value entry however requires tab or enter, which moves the focus of the text field! Which can even lead to entering play after a value change but the new value isn't applied since you haven't stopped editing it!
- Many sensible defaults are .. well, not enabled by default. For instance Rigidbody3D Contact Monitor needs to be enabled, and then their "max contacts reported" value set to 1 or higher! Awful!!

- Scene hierarchy
- you can't drop items onto the empty space below the hierarchy.


- 3D Scene view
- Multi-selection disables drawing of gizmos altogether (eg spotlight radius and range wireframes)
- Render settings affect EVERYTHING in the scene view. So if you go for 3D upscaling pixel look, all the wireframe gizmos are also pixelated! Or glow like hell!

- Extensions
- Installing an Asset Extension into a git source controlled Godot project will cause file conflicts if the Extension includes its own .git subfolder, or other common files like .gitignore and README.md. This is a problem of the Asset Library not filtering these files or allowing users to submit such extensions.
- 

- Logging / Error Messages
- Error messages, when they are C++ assertions, are meaningless. 
- For instance, this doesn't tell a user anything other than "something's wrong": Condition "array_len == 0" is true. Returning: ERR_INVALID_DATA
- Another example without context:  ERROR: Unrecognized UID: "uid://4d3xtxyvu7ky".
- If you print too much, Godot will just print:   ERROR: [output overflow, print less text!]
- can't double click log items to open the file location where it was logged from


- C#
- C# exceptions do not appear in the editor's Output window, but only in the "Debugger: Errors" tab - which loses focus every time you enter play.

- Bugs
- Delete open scene with DEL key does not delete the .tscn file nor close the open scene. Right-click => Delete however does delete the .tscn file.

- Limitations
- Renderscale doesn't go lower than 0.25
- Physics doesn't support scaled collision shapes! You have to fiddle with shape sizes, and do so every time the visual mesh scale changes. 

### Personal Opinion

## Unreal 5.6

### Pros
### Cons
### Personal Opinion


## Unity 6.0

### Pros
### Cons
### Personal Opinion
