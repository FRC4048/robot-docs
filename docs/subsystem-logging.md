# Subsystem Logging

You can treat this as a migration guide between the base AdvantageKit and our implementation.
!!! note
    this documentation assumes you are familiar with AdvantageKit

## How to log a Subsystem

Instead creating Inputs for each IO collection, a builder is used to create and an input object that has all of the desired fields.

### LoggableIO implementation

Every IO should should extend `LoggableIO<T>` where `T` is a subclass of `FolderInputs`.

Basic Motors :material-arrow-right: `MotorInputs`

Motors with PID :material-arrow-right: `PidMotorInputs`

### RealIO implementation

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

### Subsystem implementation

!!! note
    each IO collection should be in charge of at most one piece of hardware. However you may have multiple IO collection per Subsystem

In the Subsystem, IO interfaces and inputs are stored through a `LoggableSystem`. The `LoggableSystem` provides a consistent way of accessing both the IO interface's methods, and the Inputs.

``` mermaid
classDiagram
class LoggableSystem{
  +getInputs()
  +getIO()
} 
```

#### Building the Inputs

The Inputs types listed above all have a `Builder` which can be used to construct an input with different loggable attributes.

For example, if you wanted to build the subsystem from earlier:

```java
MotorInputs inputs = new MotorInputBuilder<>("FooSubsystem")
  .encoderPosition()
  .motorTemperature()
  .encoderVelocity()
  .build();
```

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

