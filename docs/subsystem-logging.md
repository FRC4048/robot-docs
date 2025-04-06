# Subsystem Logging

You can treat this as a migration guide between the base AdvantageKit and our implementation.
!!! note
    this documentation assumes you are familiar with AdvantageKit

## How to log a Subsystem

There are two methods for logging a subsystem.

The **Legacy method** involves extending `FolderLoggableInputs` and declaring the properties and how they are written and read from the log by hand.

The **recommended method** is using one of the prebuilt input classes with customizable parameters that can be selected using the corresponding Builder.

### LoggableIO implementation

Every IO should should extend `LoggableIO<T>` where `T` is a subclass of `FolderInputs`.

Basic Motors :material-arrow-right: `MotorInputs`

Motors with PID :material-arrow-right: `PidMotorInputs`

### RealIO implementation

If using the **recommended method**
The RealIO must have an `InputProvider` that matches the corresponding `Input` type.

`MotorInputs` :material-arrow-right: `MotorInputProvider`

`PidMotorInputs` :material-arrow-right: `PidMotorInputProvider`

Then, in the `updateInputs(T inputs)`, you need to write `inputs.process(inputProvider)`
Here is an bare bones example:

```java
public interface FooIO extends LoggableIO<MotorInputs> {

}

public class RealFooIO implements FooIO {
  private final MotorInputProvider inputProvider;
  private final SparkMax motor;

  public RealFooIO(){
    this.motor = new SparkMax(0,SparkMax.MotorType.kBrushless);
    this.inputProvider = new SparkMaxInputProvider(motor);
  }

  @Override
  public updateInputs(MotorInputs inputs){
    inputs.process(inputProvider);
  }
}
```

If using the **legacy method** you have to manually update the inputs.

``` java
@Override
public updateInputs(FooInputs inputs){
    fooInputs.value = true;
}
```

### Subsystem implementation

!!! note
    Each IO collection should be in charge of at most one piece of hardware. However you may have multiple IO collections per Subsystem

In the Subsystem, IO interfaces and inputs are stored through a `LoggableSystem`. The `LoggableSystem` provides a consistent way of accessing both the IO interface's methods, and the Inputs.

``` mermaid
classDiagram
class LoggableSystem{
  +getInputs()
  +getIO()
} 
```

#### Building the Inputs

As mentioned above there are two different ways of create inputs.

#### Legacy Method

Create a subclass of `FolderLoggableInputs` and then follow the example below to write to and from the log.

```java
public class FooInputs extends FolderLoggableInputs {

    public boolean value = false;

    public FooInputs(String key){
        super(key);
    }


    @Override
    public void toLog(LogTable table) {
        table.put("value", value);
    }

    @Override
    public void fromLog(LogTable table) {
        value = table.get("value", value);
    }
}
```

#### Recommended Method

The Inputs types listed above all have a `Builder` which can be used to construct an input with different loggable attributes.

For example, if you wanted to build the subsystem from earlier:

```java
MotorInputs inputs = new MotorInputBuilder<>("FooSubsystem")
  .encoderPosition()
  .motorTemperature()
  .encoderVelocity()
  .build();
```

`inputs` will then be in charge of logging `encoderVelocity`, `motorTemperature`, and `encoderVelocity` under the path `LoggableInputs/FooSystem/`.

Assuming the IO is passed in via `RobotContainer`, a new `LoggableSystem` can be constructed with

```java
fooSystem = new LoggableSystem<>(io, inputs);
```

Finally, in the Subsystem's `periodic()`, you have to call `system.updateInputs()`

## How Subsystem Logging Works

Most of the code is provided by AdvantageKit,  so what I want to go over is our pipeline for creating a loggable robot project.

This pipeline consists of

- `InputProviders`
- `Inputs`
- `InputBuilders`

### InputProviders

``` mermaid
classDiagram
InputProvider <|-- MotorInputProvider
InputProvider <|-- PidMotorInputProvider
MotorInputProvider <|-- SparkMaxInputProvider
MotorInputProvider <|-- TalonInputProvider
SparkMaxInputProvider <|-- NeoPidMotorInputProvider
PidMotorInputProvider <|-- NeoPidMotorInputProvider
class InputProvider{
  <<interface>>
}
class MotorInputProvider {
  <<interface>>
  +getMotorCurrent()
  +getMotorTemperature()
  +getEncoderPosition()
  +getEncoderVelocity()
  +getFwdLimit()
  +getRevLimit()
}
class PidMotorInputProvider{
  <<interface>>
  +getPidSetpoint()
}
class SparkMaxInputProvider{
  -sparkMax: SparkMax
}
class TalonInputProvider{
  -talon: WPI_TalonSRX
}
class NeoPidMotorInputProvider{
  - neoPidMotor: NeoPidMotor
}
```

Starting off simple, all the `InputProvider` classes do is provide a common interface for retrieving motor information. This bridges the gab between different vendors as there is no common superclass.

### Inputs

Inputs contain the actual fields that are being logged.

There are two different ways of creating `LoggableInputs`.

The *Legacy* method involves extending `FolderLoggableInputs` and declaring the properties by hand and how they are written and read from the log.

The second method is using one of the prebuilt input classes with customizable parameters that can be selected using the corresponding Builder.
If using [method 2](#method-2) you can use one of the prebuilt `FolderInputs` classes.

``` mermaid
classDiagram
LoggableInputs <|-- FolderLoggableInputs
FolderLoggableInputs <|-- FolderInputs
FolderInputs <|-- MotorInputs
MotorInputs <|-- PidMotorInputs

class LoggableInputs{
  <<interface>>
  +toLog(LogTable table)
  +fromLog(LogTable table)
}

class FolderLoggableInputs {
  +folder: String
  +getFolder(): String
}
class FolderInputs {
  #process(InputProvider provider): boolean*
}
class MotorInputs{
  encoderPosition: Double
  encoderVelocity: Double
  motorCurrent: Double
  motorTemperature: Double
  appliedOutput: Double
  fwdLimit: Boolean
  revLimit: Boolean
  builder: MotorInputBuilder<?>
}
class PidMotorInputs {
  getPidSetpoint: Double
}
```

Every `FolderInputs` (and subclasses) are in charge of logging the values that were selected using the builder.

### InputBuilders

Each InputBuilder is contains methods that set a flag indicating if the user wants to log a specific value. For example `MotorInputBuilder#encoderPosition` sets a boolean to `true` indicating that the `encoderPosition` should be logged.

``` mermaid
classDiagram
MotorInputBuilder~T extends MotorInputBuilder~ <|-- PidMotorInputBuilder~T extends PidMotorInputBuilder~
class PidMotorInputBuilder ~T extends PidMotorInputBuilder~{
  +logPidSetpoint(): T 
  +build(): PidMotorInputs
}

class MotorInputBuilder ~T extends MotorInputBuilder~ {
  +logEncoderPosition(): T
  +logEncoderVelocity(): T
  +logMotorCurrent(): T
  +logMotorTemperature(): T 
  +logFwdLimit(): T          
  +logRevLimit(): T          
  +logAppliedOutput(): T     
  +reset(): T
  +addAll(): T
  +build(): MotorInputs
  ~self(): T
}
```

!!! note
    There are also getters for the boolean values to check what is being logged.

You probably noticed the strange generic types used in the class diagram. This pattern is called [Curiously recurring template pattern (CRTP)](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern). This pattern helps us create extendable builders.

Here is a quick overview.

#### CRTP

Consider two classes `Foo` and `FooBar`

``` java
public class Foo {
  private final boolean fooing;

  public Foo(boolean fooing){
    this.fooing = fooing;
  }
}

public class FooBar extends Foo {
  private final boolean baring;

  public FooBar(boolean fooing, boolean baring){
    super(fooing);
    this.baring = baring;
  }
}
```

To create these two classes you create two Builders

``` java
public class FooBuilder {
    protected boolean fooing = false;

    public FooBuilder setFooing(boolean fooing){
        this.fooing = fooing;
    }
    public Foo build(){
        return new Foo(fooing);
    }
}

public class FooBarBuilder extends FooBuilder {
    protected boolean baring = false;

    public FooBarBuilder setBaring(boolean baring){
        this.baring = baring;
    }

    public FooBar build(){
        return new FooBar(fooing, baring);
    }
}
```

This implementation would **work**. However, the users would have to call the methods in a specific order

```java
// This works
FooBar foobar1 = new FooBarBuilder()
    .setBaring(true)
    .build();
// This works (you need to cast)
FooBar fooBar2 = (FooBar)(new FooBarBuilder()
    .setBaring(true)
    .setFooing(true)
    .build());

// This will throw an error!
FooBar fooBar2 = new FooBarBuilder()
    .setFooing(true)
    .setBaring(true)
    .build();
```

Creating `fooBar1` and `fooBar2` will run without issue but creating  will compile with no issues. However, `foodBar3` will not compile because `the method setBaring(boolean) is undefined for the type FooBuilder`.

**What?** I though we created a `FooBarBuilder`. There in lies the problem. The `setFooing(boolean fooing)` method returns a `FooBuilder` which does not have the method `setBaring(boolean baring)`.

This is where generic types come into play. If `FooBuilder` can return a subclass when we call its setter methods, we can continue to use the methods defined in the subclass.

Here is the class declaration for both `MotorInputBuilder` and `PidMotorInputBuilder`

```java
public class MotorInputBuilder<T extends MotorInputBuilder<T>>
public class PidMotorInputBuilder<T extends PidMotorInputBuilder<T>> extends MotorInputBuilder<T> {
```

Both classes have a generic type `T` which is a subtype of the current class.

Each builder method returns a type `T`. Each method returns the result of calling the `self()` method which will return the current object casted to what ever the subclass created is.

```java
T self() {         
  return (T) this; 
}                  
```

For example:

```java
public T encoderPosition() { 
  logEncoderPosition = true; 
  return self();             
}                            
```

This means that if when I create an instance of `PidMotorInputBuilder` and call the super class's method `encoderPosition` I will get back the `PidMotorInputBuilder` instance.
