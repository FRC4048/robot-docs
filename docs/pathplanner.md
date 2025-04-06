# PathPlanner

This section is more of a list of tips and tricks. Just because these work now does not mean they will work in the future.
!!! note
    For PathPlanner specific documentation see [here](https://pathplanner.dev/home.html).

## Autonomous

So you want to build an autonomous sequence...

### Organization

First imagine what path you envision the robot taking. Then, split the sequence into smaller segments. These will be your `Paths`. Although the division is arbitrary, I suggest separating the sequence into paths that each have an important *action* (like pickup, shoot, etc).

Once you have created the paths, you can combine them together into an `Auto`.
!!! important
    The `Auto` tab in pathplanner can be used for visualization but all autos MUST be explicitly declared in the java code.

When you create a new `Auto` it will act as a `SequentialCommandGroup`. You can then embed other command groups and path following commands to build out your autonomous sequence.

### Loading Paths in Code

Paths are loaded when constructed. The loading time is significant so Paths should be created as soon as possible.

One way to do this is to create and store all of path following commands in a singleton

``` java
public class Paths {
    private final Command followFooCommand;

    private static Paths instance;

    public static Paths getInstance() {
        if (instance == null) {
            instance = new Paths();
        }
        return instance;
    }
    public Paths(){
        try {
            followFooCommand = AutoBuilder.followPath(PathPlannerPath.fromPathFile("fooPath"));
        } catch (IOException | ParseException e) {
            throw new RuntimeException(e);
        }
    }

    public Command getFollowFooCommand(){
        return followFooCommand;
    }
    
}
```

### Creating Autos In Code

Create a new command that extends `LoggableSequentialCommandGroup`.

To use a path following command,

``` java
new LoggableCommandWrapper(Paths.getInstance().getFollowFooCommand());
```

!!! note
    The reason we don't store the wrapped command in `Paths` is because it would not have the correct parent command by default.

## Path on the Fly

We have Helper class called `PathPlannerUtils` that aids in creating simple paths to a desired location.

There are multiple ways getting the robot from position x to position y. Here are some pros and cons of using Paths of the fly.

### Pros

- Has built in obstacle avoidance.
- Uses the same path finding PID as autonomous.
- Simple to use.

### Cons

- Has a fixed tolerance so it can only get robot to a *general* location.

!!! note
    If you need a precise location checkout [WPILib Trajectory Generation](https://docs.wpilib.org/en/stable/docs/software/advanced-controls/trajectories/trajectory-generation.html) and [SwerveControllerCommand](https://github.com/wpilibsuite/allwpilib/blob/main/wpilibNewCommands/src/main/java/edu/wpi/first/wpilibj2/command/SwerveControllerCommand.java)
