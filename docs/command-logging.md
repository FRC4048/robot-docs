# Command Logging

## How to log Commands

- When creating a standard command extend `LoggableCommand`
- When creating a `CommandGroup` extend the corresponding Logged version
    - `ParallelCommandGroup` :material-arrow-right: `LoggableParallelCommandGroup`
    - `SequentialCommandGroup` :material-arrow-right: `LoggableSequentialCommandGroup`
    - `ParallelRaceGroup` :material-arrow-right: `LoggableRaceCommandGroup`
    - `ParallelDeadlineGroup` :material-arrow-right: `LoggableDeadlineCommandGroup`
- When using a command provided by another library, you must wrap the command using <br> `LoggableCommandWrapper(Command commandToWrap)`

### Useful Built-in Commands

- `LoggableWaitCommand`: does nothing for some amount of time
- `DoNothingCommand`: Does nothing and finishes in one tick (Useful as a placeholder)

!!! warning
    Right now, command compositions do not work. Each command must be declared in a file. In other words, you can not call `.andThen` or `.withTimeout`.

!!! important
    Commands can not be reused. They can be scheduled multiple times, but they can not have multiple parents.

## How Command Logging works

!!! note
    You sadly might have to switch to light theme for the diagram to be readable.

``` mermaid
classDiagram
  Loggable <|-- LoggableCommand
  Loggable <|-- LoggableParallelCommandGroup
  Loggable <|-- LoggableWaitCommand
  LoggableCommand <|-- DoNothingCommand
  Command <|-- WaitCommand
  Command <|-- LoggableCommand
  WaitCommand <|-- LoggableWaitCommand
  ParallelCommandGroup <|-- LoggableParallelCommandGroup
  class Loggable {
    <<interface>>
    +getBasicName() String
    +setParent(Command loggable) void
  }
  class LoggableCommand{
    -String basicName
    -Command parent
    +withBasicName(String name) LoggableCommand
  }
  class LoggableWaitCommand {
    -String basicName
    -Command parent
    +withBasicName(String name) LoggableCommand
  }
  class LoggableParallelCommandGroup{
    -String basicName
    -Command parent
    +withBasicName(String name) LoggableCommand
    +LoggableParallelCommandGroup~T extends Command & Loggable~(T...commands)
  }
  class Command
  class WaitCommand
  class ParallelCommandGroup
  class DoNothingCommand
  
  note for Command "Provided my WPILib" 
  note for WaitCommand "Provided my WPILib" 
  note for ParallelCommandGroup "Provided my WPILib" 
```

### Let's unpack this

`LoggableCommand` inherits all of WPILib's `Command` functionality. However it differs by keeping track of it parent.
In many cases, a command will be logged in the root and will have have a parent, if the command is stored in a CommandGroup, then it will know who its parent is. This is important so when we log the commands they can form a tree like structure in the log file. As a result, when looking at the logs in AdvantageScope, Commands are logged in the same hierarchy that they were declared in the code.

#### The Hijacking of `toString()`

We need some way to turn the hierarchy of commands into a path. Moreover, we need to use a method that already of exists in command (you will see why later). Sadly, there are not a lot of options as we need to `Override` a method that is not used for internal shenanigans by WPILib. Luckily, all object have a `toString()` method and it appeared WPILib did not use it for any processing!

#### Let's examine our Overrode `toString()` method

```java
@Override
  public String toString() {
    String prefix = parent.toString();
    if (!prefix.isBlank()) {
      prefix = prefix.substring(0, prefix.length() - 5);
      prefix += "/";
    }
    return prefix + getBasicName() + "/inst";
  }
```

First we get the Parent's path (using `toString()`) and the remove the "/inst" suffix. The results in a path that contains the parent log path minus that actual instance addition of the parent. From this we then append the name of this command along with "/inst" (which is why we had to remove it earlier because the parent did the same thing).
Because this block of code is in all the commands we use, we get a massive tree of path names that can be logged.

This is great and all but who actually logs the commands?

### Welcome to the `CommandLoggger`

The `CommandLogger` works by subscribing to some of the events of WPILib's `CommandScheduler:

1. `onCommandInitalize(Consumer<Command> action)`
2. `onCommandFinish(Consumer<Command> action)`
3. `onCommandInterupt(Consumer<Command> action)`

On each of these events, we store the recorded state in a queue which then gets logged every tick.

## Advanced Use cases

Sometimes it is convenient to create and schedule and command within another command.

Take the following example, where we create a command that goes to a position.

```java
public void initalize(){
  LoggableCommand command = new GoToPoseCommand(drivetrain);
  command.schedule();
}
```

This will work but when we are looking for the command in AdvantageScope, it may make more sense to see the `GoToPoseCommand` listed under the command who called it.

This can be done by calling `command.setParent(this)`.

```java
public void initalize(){
  LoggableCommand command = new GoToPoseCommand(drivetrain);
  command.setParent(this);
  command.schedule();
}
```
