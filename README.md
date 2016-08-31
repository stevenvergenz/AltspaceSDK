# AltspaceVR SDK Tutorial
## Part *n*: Adjusting Your Hat

In the last section, we went through the process of creating hats that the user could wear, and were synchronized to everyone else in the room. Our implementation had a couple problems though:

1. The hat default offsets are one-size-fits-all, and don't take into account different avatars' heads. This often results in the hat clipping through a user's head, or worse, obscuring his vision. This could be done better.

2. The hat rotates with the head, but it doesn't actually move; it just pivots in place. This is a little strange. Imagine if a user were to look straight down. The hat would be sitting sideways on the back of the user's head, instead of being anchored to the avatar's "scalp".

To fix these issues, we're going to change the way that the hat's position and orientation are calculated. We're also going to add SteamVR controller support. With the controller, the user will be able to grab her hat and manually adjust its place on her head. Let's get started!


### Adding SteamVR Controller Support

This part is rather simple as the Altspace SDK already exposes this functionality to you. Simply add the `SteamVRInput` behavior to the scene. We're also going to add an alias for this behavior at the top of the file.

At line 30, just below the hat configuration object, add the following:

```javascript
17  var config = {
        ...
28  };
29  
30  var SteamVRInputBehavior = altspace.utilities.behaviors.SteamVRInput;
31
32  var sim;
```

This is the alias we discussed. Now farther down, where we add the SceneSync behavior, we're also going to add the SteamVRInput behavior:

```javascript
102 sceneSync = ...
        ...
105 });
106 // add vive wand tracking support
107 var steamInput = new SteamVRInputBehavior(false);
108 sim.scene.addBehaviors(sceneSync, steamInput);
```

Note the `false` value that's passed into the `SteamVRInputBehavior`. This tells the system to still allow cursor events from the controllers instead of blocking them, which is the default.

We now have Vive wand tracking available to our app! Now let's take advantage of that.


### Detecting Controller Button Press Events

