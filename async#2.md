# Async C# / COBOL

## Invoking

### Method returning Task
```csharp
async void Button1_Click(object sender, EventArgs e)
{
    Button1.Text = "Processing...";
    await Task.Delay(1000);
    Button1.Text = "Done";
}
```

```cobol
method-id Button1_Click(sender as object, e as type EventArgs) async-void.
    set Button1::Text = "Processing..."
    invoke await type Task::Delay(1000)
    set Button1::Text = "Done"
end method.
```

### Await on variable or field
```csharp
async void WaitForInfo(Task t)
{
    await t;
}
```

```cobol
method-id WaitForInfo(t as type Task) async-void.
    invoke await t
end method.
```

### Return type void/empty
This is a new concept in COBOL
```csharp
async void Process()
{
    var records = await FetchRecords();
}
```

```cobol
method-id Process() async-void.
    declare records = await FetchRecords()
end method.
```

## Async Task methods
Async methods can return either `void`, `Task`, `Task``1`, `ValueTask` or `ValueTask``1`. `ValueTask` is a value type wrapping a `TResult` and `Task`. Its purpose is to improve performance of methods that frequently run synchronously (do not await) and only occasionally run asynchronously (await).
```csharp
async Task InitialiseAsync()
{
    await Task.Delay(1000);
    await Task.Delay(500);
}

// ValueTask requires a reference to System.Threading.Tasks.Extensions
async ValueTask InitialiseAsync()
{
    await Task.Delay(1000);
}

async Task<string[]> FetchRecordsAsync()
{
    await Task.Delay(1000);
    return new [] { "a", "b", "c" };
}
```

Plain async Task methods only require async modifier. 
```cobol
method-id InitialiseAsync() async.
    invoke await type Task::Delay(1000)
    invoke await type Task::Delay(500)
    goback
end method.
```

Async methods that have a return value always need to specify awaitable return type.
```cobol
method-id FetchRecordsAsync yielding return-value as string occurs any async.
    invoke await Task.Delay(1000)
    set return-value to table of ("a", "b", "c")
end method.
```

## Delegates - not currently supported
Anonymous methods / inline delegates should be able to be async.
```cobol
declare handler as type Func[binary-long, type Task] =
    delegate async using x as binary-long
               returning return-value as type Task
        invoke await type Task::Delay(100)
    end-delegate
```

## Async entry points - not implemented
Later versions of C# allow async main methods. This automatically generates a standard main method that wraps the async main method.

```cobol
method-id Main(args as string occurs any) async static.
    goback
end method.

method-id Main async static.
procedure division using by value args as string occurs any
               returning return-value as type Task[binary-long].
    set return-value to 1
    goback
end method.
```

It would be nice if you could use await in procedural programs too. The checker could automatically detect the use of `await` in the procedure division and create a wrapper entry point accordingly. Not implemented.
```cobol
*> Not implemented
program-id MyProgram.
working-storage section.
01 client type HttpClient value new HttpClient().

procedure division.
    declare str = await client::GetStringAsync("http://1.1.1.1/")
    display "Downloaded " str::Length " chars"
    goback.
end program.
```

## Try, catch, finally blocks
```cobol
method-id TryCatchAsync async.
    try
        invoke await type Task::Delay(100)
        raise new Exception()
    catch
        invoke await type Task::Delay(100)
    finally
        invoke await type Task::Delay(100)
    end-try
end method.
```

## Restrictions
Await can not be used inside sync blocks. The following code should produce an error.
```cobol
sync new object()
    invoke await type Task::Delay(100)
end-sync
```

Async modifer on `iterator-id` or `property-id` should be an error. `async` is only allowed on `method-id`.

A warning could be generated if an async method does not end with *Async* or a non-async method returns with *Async*.

A warning could be generated if an async method does not use the await keyword anywhere.

## Awaitable types - not yet implemented
The await keyword is syntantic sugar for calling `GetAwaiter()` on the expression being awaited which returns an `INotifyCompletion`. This means any object can be awaited as long as it exposes a method called `GetAwaiter` which returns a valuetype or class that implements `INotifyCompletion`.

```cobol
method-id Process async-void.
    invoke await new MyAwaitable()
end method.

class-id MyAwaitable.
method-id GetAwaiter.
procedure division returning return-value as type INotifyCompletion.
    goback
end method.
end class.
```

**Edge case:** Extension methods named `GetAwaiter` should also be considered. The `
Microsoft.VisualStudio.Threading` assembly defines many of these so that various objects can be awaited such as a `TaskScheduler` or a `WaitHandle`.
