# AltspaceVR SDK Tutorial
## Part *n*: Adjusting Your Hat

In the last section, we went through the process of creating hats that the user could wear, and were synchronized to everyone else in the room. Our implementation had a couple problems though:

1. The hat default offsets are one-size-fits-all, and don't take into account different avatars' heads. This often results in the hat clipping through a user's head, or worse, obscuring his vision. This could be done better.

2. The hat rotates with the head, but it doesn't actually move; it just pivots in place. This is a little strange. Imagine if a user were to look straight down. The hat would be sitting sideways on the back of the user's head, instead of being anchored to the avatar's "scalp".

To fix these issues, we're going to change the way that the hat's position and orientation are calculated. We're also going to add SteamVR controller support. With the controller, the user will be able to grab her hat and manually adjust its place on her head. Let's get started!


### Adding SteamVR Controller Support

This part is rather simple as the Altspace SDK already exposes this functionality to you. Simply add the `SteamVRInput` behavior to the scene. We're also going to add an alias for this behavior at the top of the file.

Just below the hat configuration object, add the following:

```javascript
var SteamVRInputBehavior = altspace.utilities.behaviors.SteamVRInput;
```

This is the alias we discussed. Now farther down, where we add the SceneSync behavior, we're also going to add the SteamVRInput behavior:

```javascript
// add vive wand tracking support
var steamInput = new SteamVRInputBehavior(false);
sim.scene.addBehavior(steamInput);
```

Note the `false` value that's passed into the `SteamVRInputBehavior`. This tells the system to still allow cursor events from the controllers instead of blocking them, which is the default.

We now have Vive wand tracking available to our app! Now let's take advantage of that.


### Cleaning Up The Follow Behavior

Calling the `FollowBehavior` function currently returns a new behavior that you're to add to the hat. This is what's called a "factory function", because the function builds new objects and returns them, instead of being the object itself which is more common in the object-oriented model. And while this factory model works in this case, the behavior as-written wouldn't work if there were more than one in the scene. So for best practices' sake, we're going to change this behavior to follow the OO pattern.

Rename the function to `FollowGrabBehavior`, and make all the variables declared at the top there into instance variables by assigning them to `this`.

```javascript
function FollowGrabBehavior(config)
{
    this.joint = config.joint;
    this.offset = config.offset;
    this.object3d = null;
    this.sync = null;
}
```

Next, take each of the functions defined on the object, and add them to the prototype. Also change all the references to the previous variables to their instance equivalents (e.g. `object3d` to `this.object3d`).

```javascript
FollowGrabBehavior.prototype.awake = function(o)
{
    this.object3d = o;
}

FollowGrabBehavior.prototype.update = function()
{
    ...
}
```

Don't spend too much time cleaning up the `update` function, because we'll be completely rewriting it in the next section, so let's just continue for now.

Make sure to also remove the line that goes `return { awake: awake, update: update };`, as it doesn't apply to the new model

Great, the behavior has been updated! Now just change the invocation to match in the `createHat` function:

```javascript
obj.addBehavior( new FollowGrabBehavior({offset: offset, joint: head}) );
```

Now let's really get to the fun part!


### Detecting Grab Events

The basic idea here is to check the controller status every so often, and see if the button state (pressed or not pressed) has changed. When the button becomes pressed, we're gonna lock the hat to the user's hand instead of his head, until they release the button. To do this, we need to add three new instance variables to the behavior's constructor. These will help us maintain an accurate picture of what the controller is doing over multiple updates.

```javascript
// stores the button state as of last check
this.grabbing = false;

// stores the position and orientation of the controller as of last check
this.inputMat = new THREE.Matrix4();

// a reference to the global SteamVRInput behavior
this.input = sim.scene.getBehaviorByType('SteamVRInput');
```

We're also going to rename our old variable `offset` to something more descriptive, and convert into a format that's more useful:

```javascript
this.offsetMat = new THREE.Matrix4().setPosition(config.offset);
```

Now let's go back down to the `update` function, and poll the controller for its state.

```javascript
// check if user is grabbing something
var grabHand;
if( this.input && this.input.leftController
    && this.input.leftController.buttons[SteamVRInputBehavior.BUTTON_GRIP].pressed
)
    grabHand = this.input.leftController;
else if( this.input && this.input.rightController
    && this.input.rightController.buttons[SteamVRInputBehavior.BUTTON_GRIP].pressed
)
    grabHand = input.rightController;
else
    grabHand = null;
```

First we declare a new variable, `grabHand`, that will store a reference to the controller that the user is grabbing with, where a grab is done by pressing the grip on the controller. This is important because Vive users have two controllers, and they could grab it with either one, so we have to check. Finally, note that if neither controller is grabbing, `grabHand` is set to `null`.

For the sake of compatibility, let's further define a grab as only counting if the controller is touching the hat. We do this by comparing the controller's position with the bounding box of the hat:

```javascript
var bounds = new THREE.Box3().setFromObject(this.object3d);
if(grabHand && bounds.containsPoint( inputPos = new THREE.Vector3().copy(grabHand.position) ))
{
    ...
}
```

After computing the bounding box using the `THREE.Box3` class, we check that the grip is being pressed, and if so, we see if the controller's position is contained inside the bounds of the hat. And since we'll need that position value again later, we save it to the `inputPos` variable.

If the user is indeed grabbing the hat, the first thing we want to do is update the controller position and orientation.

```javascript
if(...)
{
    // get controller orientation
    var inputQuat = new THREE.Quaternion().copy(grabHand.rotation);
    
    // generate a matrix from the controller position and orientation
    this.inputMat.compose(inputPos, inputQuat, new THREE.Vector3(1,1,1));

    ...
}
```

The math here can be hard to follow, but here's the bottom line: a quaternion is a structure that describes a rotation, or in this case an orientation. We combine that with the controller position that we stored before, and a scale of 1, to get the full "transform" of the controller. This transform is everything we need to know about the controller in the 3d space.

Once we've updated the controller transform, we compare the grab state this frame with the grab state last time we checked to determine if the user has only just now grabbed it. If that's the case, we convert the hat's transform relative to the user's head into a transform relative to the controller.

```javascript
if(!this.grabbing)
{
    this.grabbing = true;
    
    var handInverse = new THREE.Matrix4().getInverse(this.inputMat);
    this.offsetMat.multiply(this.joint.matrix).multiply(handInverse);
}
```

This is the same technique that's used by Three.js to implement the scene hierarchy, so this code is equivalent to changing the hat's "parent" from the head to the controller. We're not doing that though, so that we don't have to keep different scene hierarchies for local and remote versions of the hat.

We now know when the user starts grabbing the hat, but we still need to figure out when they let go. In this case, a release is when the grip button is no longer pressed, but was pressed last frame. When this happens, we need to convert the hat's transform back to being relative to the user's head. We do that by performing the opposite operation as before:

```javascript
if( /* grabHand && hand on hat */ ){
    ...
}
else if(this.grabbing)
{
    this.grabbing = false;
    
    // translate hat position/rotation from controller space back to head space
    var headInverse = new THREE.Matrix4().getInverse(this.joint.matrix);
    this.offsetMat.multiply(this.inputMat).multiply(headInverse);
}
```

This is all we need to do for detecting the grab, good job! We know when the user grabs and when she lets go, and perform the necessary operations to change the hat's anchor. But how do we use all this to actually position the hat? Read on!


### Placing The Hat

There's been a lot of matrix math up to this point, and this is where it all pays off. First, we determine if the hat should be anchored to the user's head or controller. Note the `.clone()` on each matrix. We do this because those matrixes are used on subsequent checks to the controller, and we don't want to mess them up.

```javascript
// position hat relative to head, or hand if grabbing
if(this.grabbing)
    var mat = this.inputMat.clone();
else
    var mat = this.joint.matrix.clone();
```

Once we know where the hat is anchored, we apply our previously computed offset. We also add the scale factor that we obtained from the call to `altspace.getEnclosure()` in the last lesson to make the hat its proper size:

```javascript
mat.multiply(this.offsetMat);
mat.multiply(new THREE.Matrix4().makeScale(scale, scale, scale));
```

Finally, we set the hat's transform to this new matrix that we've computed, and update its other properties based on this matrix:

```javascript
this.object3d.matrix.copy(mat);
this.object3d.matrix.decompose(object3d.position, object3d.quaternion, object3d.scale);
```

And that's it! We're done! The hat will follow the user's head unless it is being repositioned with the controller.


### Conclusion



## Appendix A: Full Source Code

```javascript
function FollowGrabBehavior(config)
{
    this.offsetMat = new THREE.Matrix4().setPosition(config.offset);
    this.joint = config.joint;
    this.grabbing = false;
    this.inputMat = new THREE.Matrix4();

    this.object3d = null;
    this.input = sim.scene.getBehaviorByType('SteamVRInput');
}

FollowGrabBehavior.prototype.awake = function(o)
{
    this.object3d = o;
}

FollowGrabBehavior.prototype.update = function()
{
    // check if user is grabbing something
    var grabHand;
    if( this.input && this.input.leftController && this.input.leftController.buttons[SteamVRInputBehavior.BUTTON_GRIP].pressed )
        grabHand = this.input.leftController;
    else if( this.input && this.input.rightController && this.input.rightController.buttons[SteamVRInputBehavior.BUTTON_GRIP].pressed )
        grabHand = input.rightController;
    else
        grabHand = null;

    // check if user is grabbing the hat
    var inputPos, inputQuat;
    var bounds = new THREE.Box3().setFromObject(this.object3d);
    if(grabHand && bounds.containsPoint( inputPos = new THREE.Vector3().copy(grabHand.position) ))
    {
        inputQuat = new THREE.Quaternion().copy(grabHand.rotation);

        // update controller position
        this.inputMat.compose(inputPos, inputQuat, new THREE.Vector3(1,1,1));

        // grab
        if(!this.grabbing)
        {
            this.grabbing = true;

            // translate hat position/rotation from head space to hand space
            var handInverse = new THREE.Matrix4().getInverse(this.inputMat);
            this.offsetMat.multiply(this.joint.matrix).multiply(handInverse);
        }
    }

    // release
    else if(this.grabbing)
    {
        this.grabbing = false;

        // translate hat position/rotation from controller space back to head space
        var headInverse = new THREE.Matrix4().getInverse(this.joint.matrix);
        this.offsetMat.multiply(this.inputMat).multiply(headInverse);
    }


    //No need to take ownership of this hat, since we created it.

    // position hat relative to head, or hand if grabbing
    if(this.grabbing)
        var mat = this.inputMat.clone();
    else
        var mat = this.joint.matrix.clone();

    // offset
    mat.multiply(this.offsetMat);

    // scale
    mat.multiply(new THREE.Matrix4().makeScale(scale, scale, scale));

    // apply
    this.object3d.matrix.copy(mat);
    this.object3d.matrix.decompose(object3d.position, object3d.quaternion, object3d.scale);
}

```