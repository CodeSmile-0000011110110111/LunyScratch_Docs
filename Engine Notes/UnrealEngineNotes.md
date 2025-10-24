# Engine Notes

Observations, complaints, niceties.


## Unreal 5.6


### Pros

- UX
- you work on a grid by default, so very welcome!
- live-editing blueprints while in playmode, yeah!
- looks good out of the box

### Cons

- CRASH!!! Lots and lots of crashes, with loss of data :(
  - ie edit some blueprint values while in playmode => prone to crash
  - change a light? change a sound? change a setting? crash, crash, crash
  - change a blueprint's base class? crash! (known issue since 2016 => won't fix)
  - data isn't saved until you "save all", including newly imported assets!! => Unreal trains you to save, save, save ...

- UX
- Launcher and dialog windows pop up in the background, including things like "Do you want to .. ?  YES/NO" which is confusing if that dialog is not shown to me.
- progress is shown in notification widgets, which can mean a huge stack of such widgets for complex tasks. I completely overlooked a "restart required" notification, since it's tugged in the very opposite corner I work with in the editor.
- The "Delete xxx" dialog box has no focus, so you can't DEL and RETURN to quickly confirm deletion.
- Can't delete assets until after "save all" - it simply doesn't do anything if there's unsaved changes!
- Saving/deleting even a "blank" level takes much longer (3-5 seconds) than in other engines, where it is instant for "blank" scenes.
- Lost imported assets due to another instance of the project having been launched, and stuck at the "previous import failed" (or similar) message which caused all meshes to lose their textures.
- Shift+F1 to "free the mouse" is such an awkward hotkey - and it cannot be changed!!
- Save All means "save all, no questions asked" but in Unreal it means: okay, which of these files do you wish NOT to be saved? Darn ...
- Lighting/visual tweaks: it takes 10+ seconds for lighting to adapt to new settings, and on every change it resets to the base level as the camera slowly adapts to the current lighting level (camera exposure or whatever). Perhaps possible to adjust but not the default.
- Content editing is the least intuitive among Godot and Unity.
- empty scene (with terrain) takes ~20 seconds to save

- Assets
- drag and drop of folder with contents opens up an import dialog where you can change settings, rather than allowing me to do that after the fact
- FBX are imported as individual objects by default, so instead of Kenney's police car I have to choose which four wheels I need, which car body, and which grill. => fixed by checking "Combine Static Meshes"
- delete assets may pop up window telling me that there's still references "in memory" (what i dropped into the scene). It asks me with a big red button whether i want to "Force Delete".
- all of a sudden all my meshes went black. I didn't even have the editor focused, I just noticed the thumbnails getting redrawn.
- Assets are binary :(
- It appears unless meshes are imported into subfolders one runs the risk of overwriting a common asset such as colormap, which means all assets look incorrect.
