# Autonomously Calculating Shooting Angle

There are two approaches to calculating a desired shooting angle:

1. Interpolate between measured data points.
2. Physics Based simulation.

Both have their pros and cons:

## Why Use Interpolation

This method should be used when you have a complex trajectory that is hard to model with physics equations. If your trajectory is parabolic or linear you might want to consider using a [Physics Model](#why-use-a-physics-model).

The main *advantage* of Interpolation is its simplicity.
The main *disadvantage* of Interpolation is that the robot must be stationary.

## Why Use a Physics Model

This method should be used when your trajectory is simple examples include 2022 game Rapid React. Or the 2024 game Crescendo.

In Rapid React, the ball game piece were simple to model with physics equations.

In Crescendo, teams fired the note so quickly that the complicated drag of a torus did not have any effect on the trajectory.

The main *advantage* of a Physics Model is that it works when the robot is moving.
The main *disadvantage* is that it requires math with ranging difficulty

## Interpolation Implementation

WPILib provides a class called [InterpolatingDoubleTreeMap](https://github.wpilib.org/allwpilib/docs/release/java/edu/wpi/first/math/interpolation/InterpolatingDoubleTreeMap.html). What it allows us to do is provide a list of measured angles at specific distances from the target, and then estimate what the current angle should be based on the current distance's proximity to other measurements.

For example, consider the following measurements in the format (distance from target, angle to shoot):
(1 meters, 50 degrees)
(2 meters, 40 degrees)

If I then want to know what angle to shoot at when we are 1.5 meters away, the `InterpolatingDoubleTreeMap` would tell us that we have to shoot at 45 degrees (half way between 50 and 40).

## Physics Model Implementation

!!! note
    Because the Physics model can be game piece dependant. I am going to go over an example without modeling *drag* or *spin*.

### Projectile Motion

!!! important
    For a deeper explanation of the math see the [presentation](https://docs.google.com/presentation/d/1C3Zrz4nMrskZdTj3XkqiU0yrXnFCWz1SGjXfObQNPAk/edit?usp=sharing)
Here is the equation for calculating shooting angle

$$
\theta = \arctan{\frac{v^2 \pm \sqrt{v^4 - g(gx^2 + 2yv^2)}}{gx}}
$$

- $\theta$ represents angle to shoot
- $v$ represents game piece initial velocity (combined x and y)
- $g$ represents gravity
- $x$ represents the x distance from the target (left to right)
- $y$ represents the y distance from the target (up and down)

It is important to note that this equation is for 2D space. "But we are in 3D I hear you ask". Well no worries. We can convert our 3D environment in 2D by rotating the robot so it faces the target.

Given our 3D coordinates (x,y,z) we combine x and y into one variable $x^\prime$ and our z value becomes $y^\prime$

Now if we take find the distance in the xy plane.

$(x,y,z)$ :material-arrow-right: $\sqrt{x^2 + y^2}$ :material-arrow-right: $y^\prime$

$(x,y,z)$ :material-arrow-right: $z$ :material-arrow-right:$y^\prime$

Where $(x^\prime, y^\prime)$ represent $(x,y)$ in 2D space.

Here is code Implementation

``` java
public static double getXYDistance(Pose3d pose3d){
    return xyDist = Math.hypot(pose3d.getX(), pose3d.getY());
}
```

That was the most roundabout way of explaining the Pythagorean theorem but hopefully you understand.

Now that you know how to get the 2D x distance and the y distance. You now need to know the game piece's initial velocity.

You can find this by recording a slow motion video with a phone and counting the number of frames for the game piece to move some small distance. If the slow motion is not slow enough, you can ask to barrow one of the motion gates from Mr. Daudelin (high school physics teacher). In 2024 we used a Samsung S22 Ultra which recorded at 960 fps replayed at 30fps.

That means that $\frac{1~real~second}{960~frames} \times \frac{30~frames}{1~playback~second} = \frac{30~real~seconds}{960~playback~seconds}$ :material-arrow-right: $1~playback~second = 0.03125~real~seconds$

Now you can use $x=vt$ to solve for v and get an initial velocity in $\frac{meters}{second}$

Congratulations you have everything you need to run a basic physics simulation!

Now it would be rude to make you code all of that yourself, so I have created some utility classes to help you.

### Vector Utils

This class generates `VelocityVector`.

A `VelocityVector` has a `double velocity` and a `Rotation2d angle`

#### horizontalRange function

This function takes in a `VelocityVector` and returns how far the projectile would theoretically go before hitting the ground.

This function can be used to check what portion of the trajectory you are hitting your target. For example in Crescendo, we said we could only shoot if the trajectory arrived at the target in the first 2/5 of the trajectory. This was to prevent hitting the target from above because the target have a roof we had to shoot under.

``` java
public static double horizontalRange(VelocityVector velocityVector) {
    ... // Implementation Hidden
}
```

#### fromVelAndDist function

This function generates a `VelocityVector` from an initial `speed`, an `deltaX` from the target, a `deltaY` from the target and a `maxFractionalRange` (used in the [horizontal range function](#horizontalrange-function))

This methods implements a variation of the math [above](#physics-model-implementation)

``` java
public static VelocityVector fromVelAndDist(double speed, double deltaX, double deltaY, double maxFractionalRange) {
    ... // Implementation Hidden
}
```

Before we get into the other methods we need to talk about *Shooting while moving*.

One of the biggest selling points with using a physics simulation is that it allows us to shoot while moving yet our equations does not take into account the robots velocity at all! What is up with that?

Before I answer that Let's Take a step back and look at how movement effects scoring.

### Shooting While Moving

When the robot is moving it has some `xVelocity` and some `yVelocity` the game piece then *inherits* the robotics velocity towards its initial shooting velocity.

The portion of the robot's velocity that impacts the game piece's velocity can be found by calculating what portion of the velocity moves the robot closer or farther away from the target. That is to say movement around the target in 3D space what does not get the robot closer to the target is not important to the game piece's velocity.

Here is how we find that velocity

1. Subtract target position vector from robot position vector and normalize it
2. Take the dot product of the robotics velocity vector and the normalized vector.
Lets call this our `i-velocity`
!!! note
    The `i-velocity` was not properly implemented in 2024's game Crescendo.

First we need to find what portion of the `i-velocity` impacts the game piece's velocity. The portion of the robots velocity that effects

This is problematic because we have to add the `i-velocity` vector with the game piece's initial velocity vector however we don't know the angle we are shooting at yet!

In other words we cant add the vectors until we know the angle we are shooting and we can't find the angle we are shooting until we add the vectors.

Well this stinks.

Luckily I have a solution.

Guess!

What?

You heard me, guess.

#### fromDestAndCompoundVel

Let's guess we are shooting at a `45 degree` angle.

This then allows us to add our `i-velocity` to our initial velocity.

``` java
double xySpeed = (Math.cos(45) * speed) + driveSpeedX;
double zSpeed = Math.sin(45) * speed;
```

We can then combine these speeds into a new initial velocity of the game piece.

```java
double appliedSpeed = Math.hypot(xySpeed, zSpeed);
```

Finally, we calculate the shooting velocity by calling [`fromVelAndDist`](#fromvelanddist-function)

But our solution is wrong you might ask. And you would be correct.

The solution is **TO DO IT AGAIN**. Except this time use the angle we got from the previews step to add the `i-velocity` to the initial piece velocity.

You can keep repeating this to get more and more accurate results.

!!! note
    5 iterations is already overkill. You will be bottle necked by the mechanical accuracy long before software.

Hey look that that they iteration method already exists~

``` java

public static VelocityVector fromDestAndCompoundVel(double speed, double startX, double startY, double startZ, double driveSpeedX, double destX, double destY, double destZ,double distOffset ,double degreeThreshold, int maxIterations, double maxFractionalRange) {
    ... // Implementation Hidden
}

```
