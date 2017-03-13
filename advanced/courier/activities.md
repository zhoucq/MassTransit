# Activities

In MassTransit Courier, an *Activity* refers to a processing step that can be added to a routing slip. 
To create an activity, create a class that implements the *Activity* interface.

```csharp
public class DownloadImageActivity :
    Activity<DownloadImageArguments, DownloadImageLog>
{
    Task<ExecuteResult> Execute(ExecutionContext<DownloadImageArguments> context);
    Task<CompensationResult> Compensate(CompensateContext<DownloadImageLog> context);
}
```

The *Activity* interface is generic with two arguments. The first argument specifies the activity’s input type and 
the second argument specifies the activity’s log type. In the example shown above, *DownloadImageArguments* is the 
input type and *DownloadImageLog* is the log type. Both arguments must be interface types so that the implementations 
can be dynamically created.

## Execute Activities

An *Execute Activity* is an activity that only executes and does not support compensation. As such, the definition 
of a log type is not required.

```csharp
public class ValidateImageActivity :
    ExecuteActivity<ValidateImageArguments>
{
    Task<ExecuteResult> Execute(ExecutionContext<DownloadImageArguments> context);
}
```

## Implementing an activity

An activity must implement two interface methods, *Execute* and *Compensate*. The *Execute* method is called while 
the routing slip is executing activities and the *Compensate* method is called when a routing slip faults and needs 
to be compensated.

When the *Execute* method is called, an *execution* argument is passed containing the activity arguments, the routing 
slip *TrackingNumber*, and methods to mark the activity as completed or faulted. The actual routing slip message, 
as well as any details of the underlying infrastructure, are excluded from the *execution* argument to prevent 
coupling between the activity and the implementation. An example *Execute* method is shown below.

```csharp
async Task<ExecutionResult> Execute(Execution<DownloadImageArguments> execution)
{
    DownloadImageArguments args = execution.Arguments;
    string imageSavePath = Path.Combine(args.WorkPath, 
        execution.TrackingNumber.ToString());

    await _httpClient.GetAndSave(args.ImageUri, imageSavePath);

    return await execution.Completed(new DownloadImageLogImpl(imageSavePath));
}
```

Once activity processing is complete, the activity returns an *ExecutionResult* to the host. If the activity executes successfully, the activity can elect to store compensation data in an activity log which is passed to the *Completed* method on the *execution* argument. If the activity chooses not to store any compensation data, the activity log argument is not required. In addition to compensation data, the activity can add or modify variables stored in the routing slip for use by subsequent activities.

> In the example above, the activity creates an instance of a private class that implements the *DownloadImageLog* 
> interface and stores the log information in the object properties. The object is then passed to the *Completed* method 
> <for storage in the routing slip before sending the routing slip to the next activity.

## Compensating an activity

When an activity fails, the *Compensate* method is called for previously executed activities in the routing slip that stored compensation data. If an activity does not store any compensation data, the *Compensate* method is never called. The compensation method for the example above is shown below.

```csharp
Task<CompensationResult> Compensate(Compensation<DownloadImageLog> compensation)
{
    DownloadImageLog log = compensation.Log;
    File.Delete(log.ImageSavePath);

    return compensation.Compensated();
}
```

Using the activity log data, the activity compensates by removing the downloaded image from the work directory. Once the activity has compensated the previous execution, it returns a *CompensationResult* by calling the *Compensated* method. If the compensating actions could not be performed (either via logic or an exception) and the inability to compensate results in a failure state, the *Failed* method can be used instead, optionally specifying an *Exception*.


