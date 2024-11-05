---
title: "Event Handling in PowerShell with C# Managed Code"
date: 2024-11-04 19:35:00 
categories: [PowerShell]
tags: [powershell, csharp, event-handling, automation]
item: assets/images/post-01-01.png
---

## Introduction
Integrating C# code within PowerShell can extend its capabilities significantly. By embedding custom C# classes, PowerShell can handle more complex operations such as event handling with a higher degree of control. 

In this post, I will walk you through a straightforward example of using pure C# code to instantiate a Timer object, expose its lapse event, and demonstrate how to subscribe to this event with a custom event handler in PowerShell.

## The Embedded C# Code
Let's start by embedding C# code in PowerShell. 

This C# snippet defines a ProcessMonitor class that triggers an event every second, incrementing a counter each time:

```powershell
$code = @"
using System;
using System.Timers;

namespace PowerShellEventDemo
{
    public class ProcessEventArgs : EventArgs
    {
        public string ProcessInfo { get; set; } // Holds process info
        public int Counter { get; set; } // Counter value
    }

    public class ProcessMonitor
    {
        public event EventHandler<ProcessEventArgs> TimeChanged; // Event to notify when timer elapses
        private int _counter = 0;
        private static Timer timer; // Timer instance

        public void Start()
        {
            timer = new Timer { Interval = 1000, AutoReset = true, Enabled = true }; // Timer setup
            timer.Elapsed += Timer_Elapsed; // Attach event handler
        }

        private void Timer_Elapsed(object sender, ElapsedEventArgs e)
        {
            _counter++; // Increment counter

            // Whenever the timer object emits an event it will raise our custom OnTimeChanged event that will be subscribed by our PowerShell code

            OnTimeChanged(new ProcessEventArgs // Raise event
            {
                ProcessInfo = String.Format("Counter: {0} Time: {1}", _counter, e.SignalTime),
                Counter = _counter
            });
        }

        protected virtual void OnTimeChanged(ProcessEventArgs e)
        {
            TimeChanged.Invoke(this, e); // Invoke event
        }

        public void Stop()
        {
            timer.Stop(); // Stop timer
        }
    }
}
"@

Add-Type -TypeDefinition $code  # Add the defined C# code to the current PowerShell session

```

## Setting Up Event Handling in PowerShell

With the C# class added to our PowerShell session, we can create an instance of ProcessMonitor and register an event handler using Register-ObjectEvent:

```powershell
# Creates an instance of the `ProcessMonitor` class defined in the embedded C# code
$monitor = New-Object PowerShellEventDemo.ProcessMonitor 

# Registers an event handler for the `TimeChanged` event, specifying the action to take when triggered
Register-ObjectEvent -InputObject $monitor -EventName TimeChanged -Action {
    
    # Extracts the event arguments passed to the handler
    $eventArgs = $Event.SourceEventArgs 
    
    Write-Host "PowerShell received: $($eventArgs.ProcessInfo)"

    # Checks if the counter has reached 10 and triggers the stop process 
    if ($eventArgs.Counter -eq 10) { 
        Write-Host "Counter: $($eventArgs.Counter) - Stop" -ForegroundColor Yellow
        $monitor.Stop() 

        # Unregisterr the event to prevent memory leak and remove the job
        Get-EventSubscriber | Unregister-Event
        Get-Job | Remove-Job
    }
}

# Start the timer
$monitor.Start() 
```

Running the code you should see the events being output to the console as the timer triggers each second:

![PowerShell Output](assets/images/post-02-01.gif){: .normal }


## Conclusion
Embedding C# code in PowerShell scripts lets you take advantage of event handling, making it possible for automation tasks to respond dynamically to specific conditions. This example shows how PowerShell’s flexibility can be combined with the power of C# to create more advanced scripting solutions.

I hope you enjoyed this post and learned something new about PowerShell!

## References
1. PowerShell and Events
   - <https://learn-powershell.net/2013/02/08/powershell-and-events-object-events/>
2. Executing C# code using PowerShell script
   - <https://blog.adamfurmanek.pl/2016/03/19/executing-c-code-using-powershell-script/index.html>
3. Timer.Elapsed
   - <https://learn.microsoft.com/en-us/dotnet/api/system.timers.timer.elapsed?view=net-8.0>
