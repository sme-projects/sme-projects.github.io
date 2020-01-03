# SME: Synchronous Message Exchange

SME is a runtime environment that allows you to develop, test and verify circuits for FPGA devices in C# with access to modern debugging and code features.

## Introduction
Programs written for a traditional CPU, use a sequential model where each  line (or instruction) is executed before proceesing to the next, giving a programmer an intuition for what the program does.

A fully sequential program would need to be implemented on an FPGA as a single long circuit path. This would significantly reduce the acheived speed a area utilization of an FPGA.

On the other hand, a fully concurrent program would be extremely difficult to program and understand.

To balance these two extremes, SME builds on a formal algebra for expressing concurrent programs without any chance for deadlocks or race conditions normally associated with concurrent programming.

## SME basics
There are only two basic components in SME: processes and busses.

A process is a self-contained unit comprising code and local data. The process contains only sequential code and can be tested and verified fully isolated.

A process can only communicate with other processes through a communication channel. In SME we group communication channels into busses, as we often need to send multiple values between processes.

The busses allow modelling of the propagation delays found in FPGAs making is easy to follow and debug for a software programmer.

Since processes only communicate through busses, we can easily determine which processes depend on others and thus extract the maximally allowed concurrency from the _structure_ of a program. SME does *not* attempt to _auto parallelize/vectorize_  your code.

This leaves full transparency for the programmer with a "what-you-write-is-what-you-get" without hidden costs or quirks.

The encapsulation of processes also allows for full component re-use and supports a compositional design method where you can either build small components and string them together, or build a large component and gradually split it until it has the desired granularity.


## SME implementation
SME itself is a programming model that can be implemented in any language, but in here we describe it in terms of the C# implementation of SME.

Below we walk through a basic example where we implement a process that increments a number.

### The input bus
To write a program in SME, you usually start by defining a bus for input and a bus for output. If you were writing a function, you could consider this the inputs and outputs of a function. An example could be:

```csharp
public interface IIncrementerInput : IBus
{
    [InitialValue(false)]
    bool Valid { get; set; }
    int NextValue { get; set; }
}
```

When working with FPGA designs we do not have the concept of `null` because the implementation will end up with circuits and we cannot have a circuit that is not connected. For this reason we often need to have an extra signal that conveys information about the presence of the value. This extra signal is usually called `Valid` but can be anything you like.

The attribute `[InitialValue(false)]` marks the signal as being reset to `false`. This is required as signals by default have an undetermined value (static noise on the connections) and we do not allow a program to read this value. By forcing a value, we can safely read it and be assured that it always holds a valid value.

The `NextValue` signal is used to send the value that we want the counter to start from. This value is only read when `Valid` is `true` and we expect the writer of this process to not set `Valid` before setting `NextValue`.

### The output bus
A process is usually not of any value unless it produces an output. For the simple incrementer we could define a bus such as this:

```csharp
public interface IIncrementerOutput : IBus
{
    [InitialValue(false)]
    bool Valid { get; set; }
    int IncrementedValue { get; set; }
}
```

This follows the same layout as the input, and we use the same logic with a `Valid` flag that allows any consumers of the data to know if they can read `IncrementedValue` or not.

### The incrementer process
The process that increments the number is very simple, but we still need to wire up the inputs and outputs:

```csharp
class MyIncrementer : SimpleProcess
{
    readonly IIncrementerInput m_input;
    readonly IIncrementerOutput m_output;

    public MyIncrementer(IIncrementerInput input, IIncrementerOutput output) 
    {
        m_input = input;
        m_output = output;
    }

    protected override void OnTick()
    {
        m_output.Valid = m_input.Valid;
        if (m_input.Valid)
            m_output.IncrementedValue = m_input.NextValue + 1;
    }
}
```

In this example we have local storage for the bus connections that we require. The constructor is used to wire up these two instances.

The main work is done in the `OnTick()` method. We mimic the `Valid` flag, such that our output is only valid when the input is valid. When we have an input, we output the incremented value.

Since we do not always write the output (when there is no input) the output retains its previous value, but we still set `Valid` to false. This will automatically create a `latch` in the output, so we can keep the value.


### Writing a simulator / test
Now that we have an incrementer, we need to somehow ensure ourselves that it actually works as expected. Since SME is written in C# we can leverage anything we need to simulate correcly, such as file access, databases, web content etc.

For the test of the incrementer we want to verify that it sends out the incremented value, and also that it returns the latched value, even if `Valid` is false.

```csharp
class MyTester : SimulationProcess
{
    readonly IIncrementerOutput m_input;
    readonly IIncrementerInput m_output;

    public MyIncrementer(IIncrementerOutput input, IIncrementerInput output) 
    {
        m_input = input;
        m_output = output;
    }

    public Task RunAsync()
    {
        await ClockAsync();

        Assert.Equal(m_input.Valid, false);

        for (var i = 0; i < 10; i++)
        {
            m_output.Valid = true;
            m_output.Value = i;

            await ClockAsync();

            Assert.Equal(m_input.Valid, true);
            Assert.Equal(m_input.Valid, i + 1);
        }

        m_output.Valid = false;
        await ClockAsync();
        Assert.Equal(m_input.Valid, 9 + 1);
    }
}
```

