# Technical Information and Experimental Results on WorkerService and MyServiceBase

Index

 [1. Introduction](#1introduction)  
 [2. Target Audience](#2target-audience)  
 [3. Notes](#3notes)  
 [4. Key Points (Conclusion)](#4key-points-conclusion)  
 [5. System Behavior (Overview)](#5system-behavior-overview)  
 [6. Differences Between .NET 10.0 and .NET Framework 4.8](#6differences-between-net-100-and-net-framework-48)  
 [7. Considerations (Summary)](#7considerations-summary)  
 [8. Example Logs from MyServiceBase Execution](#8example-logs-from-myservicebase-execution)  
 [A1. Appendix — Experimental Attempts and Results (Overview)](#a1appendix---experimental-attempts-and-results-overview)  
 [S. Author](#sauthor)

---

## 1.Introduction

Handling events from the Service Control Manager (SCM) when creating a Windows Service—especially the **PreShutdown** event—is notoriously difficult.

In the past, under the .NET Framework, this could be handled relatively easily by inheriting from the `ServiceBase` class.  
However, with the `.NET 10.0 (.NET Core family)` + `WorkerService` model, the situation is very different, and handling these events has become significantly more challenging.

Furthermore, the following explanations are commonly found online:

- “WorkerService automatically creates a `ServiceBase` internally when `.UseWindowsService()` is called.”
- “You can still receive SCM events by creating a class that inherits from `ServiceBase` in WorkerService.”

However, based on the actual experiments conducted using the WindowsService .NET library for WorkerService,  
**these explanations do not accurately reflect the real behavior**.

They are not entirely false, but **none of them work when attempting to capture the PreShutdown event**.  
For users who specifically need to handle PreShutdown, this missing information is critical.

In this document, using Windows 11 Pro 24H2, I explain:

- How `ServiceBase` actually behaves under the WorkerService model  
- How to correctly receive `PreShutdown` and `Shutdown` events in WorkerService  
- The actual experimental results and logs  

I hope this information will be helpful for those who want to handle SCM events in WorkerService.

## 2.Target Audience

- Developers who want to receive the `PreShutdown` event in `.NET 10.0 WorkerService`  
- Developers who want to understand what happens when using the WindowsService class library’s `ServiceBase` within the `.NET 10.0 WorkerService` model  
- Developers who want to understand the behavioral differences between `.NET Framework 4.8` and `.NET 10.0`  

---

## 3.Notes

The experiments described in this document were conducted under the following environment:

- Windows 11 Pro 24H2  
- C# (.NET 10.0) WorkerService  
- C# (.NET Framework 4.8) ServiceBase  
- Visual Studio 2026  
- CPU: Intel J4125 (known for relatively slow TLS initialization)  
- Network: Wired LAN  

Please note that the behavior described in this document may vary depending on  
the OS version, security updates, and the installed .NET version.

## 4.Key Points (Conclusion)

- I was able to implement a `MyServiceBase` class that preserves the `.NET 10.0 WorkerService` model while providing the same functionality as the traditional `ServiceBase` (`OnStart` / `OnStop` / `OnShutdown`), and additionally supports **`OnPreShutdown`**, which does not exist in the original `ServiceBase`.

- It is true that the `.UseWindowsService()` method included in the *Microsoft.Extensions.Hosting.WindowsServices* package internally creates a `ServiceBase` instance.  
  However, **there is no mechanism for the user to configure or control this `ServiceBase`**.

- It is technically possible to create a class that directly inherits from `ServiceBase` instead of using `BackgroundService` in WorkerService.  
  However, this class **cannot configure SCM settings**, meaning it **cannot enable or receive the `PreShutdown` event**.

- In `.NET 10.0 WorkerService`, the only way to receive the `PreShutdown` event is to **directly call the Win32 APIs**:  
  `RegisterServiceCtrlHandlerEx`, `SetServiceStatus`, and `ChangeServiceConfig2`.  
  This was known as a last-resort method, but after extensive trials and measurements—where all other approaches failed—this proved to be the **only successful solution**.

---

## 5.System Behavior (Overview)

This system implements a mechanism that allows the `WorkerService` model to correctly receive  
**`PreShutdown` and `Shutdown` events from the SCM**, without breaking the WorkerService architecture.

The key structural points are as follows.

### 5.1. Worker Class

`MyWorker` uses a custom `MyServiceBase`—which itself inherits from `BackgroundService`—as its base class.

This allows the WorkerService lifecycle (`ExecuteAsync` / `CancellationToken`) to remain intact.  
Meanwhile, SCM events are obtained directly through the Windows API, and the event-handling model of the traditional `ServiceBase` (`OnXXXXXX` override methods) is exposed to the Worker class.

By overriding these methods, the user can handle SCM events with a very simple and intuitive workflow.

### 5.2. Program Startup Behavior

In `Program.cs`, the Worker is started as follows:

- `.UseWindowsService()` registers the application as a Windows Service  
- `MyWorker` is registered as a `HostedService` in the Generic Host  
- The host is built and executed

```csharp
Host.CreateDefaultBuilder(args)
    .UseWindowsService()
    .ConfigureServices(services =>
    {
        services.AddHostedService<MyWorker>();
    })
    .Build()
    .Run();
```

### 5.3. Hooking SCM Events

When the Worker starts (at the beginning of `ExecuteAsync`),  
**`RegisterServiceCtrlHandlerEx` (Win32 API)** is called to register a handler that receives control events from the SCM.

This allows the Worker to directly receive the following events:

- `SERVICE_CONTROL_STOP`  
- `SERVICE_CONTROL_SHUTDOWN`  
- `SERVICE_CONTROL_PRESHUTDOWN`

### 5.4. Enabling PreShutdown

To receive the `PreShutdown` event, the SCM must be explicitly configured with:

- `SERVICE_ACCEPT_PRESHUTDOWN`  
- A shutdown wait time (`dwPreshutdownTimeout` in milliseconds)

These are configured using:

- `SetServiceStatus` (Win32 API)  
- `ChangeServiceConfig2` (Win32 API)

Configuration details:

- Enable `SERVICE_ACCEPT_PRESHUTDOWN`  
  (Because this is a bitmask, the safe approach is to read the current value, OR the new flag, and write it back.)
- Set `dwPreshutdownTimeout` (milliseconds)

Once configured, the SCM will send the `PreShutdown` event and delay the system shutdown for the specified duration.

### 5.5. waitHint / checkPoint Updates and CancellationToken During PreShutdown

- During `PreShutdown`, the SCM periodically requests a “liveness check”.
- `MyServiceBase` automatically updates `waitHint` and `checkPoint` during this period.
- If the system issues a forced stop request, the `CancellationToken` becomes *true*.
- Even if the configured wait time has not elapsed, user code should respect this token and exit promptly.

Based on experimental results:

- Even without updating `waitHint` / `checkPoint`, the SCM still waits for the configured timeout.
- If liveness updates are performed, the SCM may wait slightly longer than the configured timeout (not fully verified).
- `MyServiceBase` is designed to wait an additional ~8 seconds beyond the configured timeout if the task does not return (effectiveness not fully verified).

If the shutdown process completes significantly earlier than the configured timeout,  
ignoring the `CancellationToken` may not cause issues (because the token fires later).  
However, since the token may also be triggered for reasons other than timeout (not fully verified),  
it should not be ignored.

### 5.6. Processing and Signal Flow (Overview)

The actual internal behavior is more complex than described here.  
A fully detailed explanation would be extremely long, so this section provides a simplified overview that remains accurate without unnecessary depth.  
For deeper understanding, please refer to specialized documentation.

### 5.6.1 Standard WorkerService Startup and Shutdown

#### 【Startup】

- The SCM launches the service executable and sets the state to `START_PENDING`.
- Inside the executable, `WindowsServiceLifetime` and the internal `ServiceBase` (from `Hosting.WindowsServices`) are initialized, preparing the bridge between the SCM and the .NET Host.
- The SCM waits for the callback registration of `ServiceBase.OnStart`, then invokes `OnStart`.
- Inside `OnStart`, the .NET Host begins executing `StartAsync`.
- `StartAsync` calls `BackgroundService.StartAsync`, which starts the user’s main logic (`ExecuteAsync`) asynchronously.
- Once `StartAsync` is invoked, `OnStart` returns, and the SCM transitions the service to `RUNNING`.
- Meanwhile, `.NET Host` continues running `StartAsync` and monitors the lifecycle of the Worker.

Key points:

- Point 1: `StartAsync` is invoked by the .NET Host, not by the SCM.  
- Point 2: Only `OnStart` is invoked as a callback; other events are delivered as control messages.  
- Point 3: As long as the Host successfully starts `StartAsync`, the SCM considers the service “started”.

#### 【STOP】

- The SCM sends `SERVICE_CONTROL_STOP` to `ServiceBase` and sets the state to `STOP_PENDING`.
- `ServiceBase.OnStop` is invoked.
- Inside `OnStop`, the .NET Host calls `StopAsync`, which triggers `StoppingToken.Cancel()`, causing the `CancellationToken` to become *true*.
- `ExecuteAsync` detects the cancellation and begins shutdown processing.
- Once `ExecuteAsync` completes, `StopAsync` completes, and `ServiceBase.OnStop` returns.
- When `OnStop` returns, the service executable exits.
- The SCM transitions to `STOPPED`.

If the executable does not exit within 30 seconds after `SERVICE_CONTROL_STOP`,  
the SCM forcibly terminates the process.

### 5.6.2 Startup and Shutdown Behavior in This System

#### 【Startup】

- The SCM launches the service executable and sets the state to `START_PENDING`.
- Inside the executable, `WindowsServiceLifetime` and the internal `ServiceBase` (from `Hosting.WindowsServices`) are initialized.
- The SCM waits for the callback registration of `ServiceBase.OnStart`, then invokes it.
- Inside `OnStart`, the .NET Host begins executing `StartAsync`.
- `StartAsync` calls `BackgroundService.StartAsync`, which starts the main logic of `MyServiceBase` (`ExecuteAsync`) asynchronously.
- Once `StartAsync` is invoked, `OnStart` returns, and the SCM transitions the service to `RUNNING`.
- Meanwhile, `.NET Host` continues running `StartAsync` and monitors the Worker lifecycle.

(Up to this point, the behavior is identical to the standard WorkerService model.)

- `MyServiceBase` calls `RegisterServiceCtrlHandlerEx` (Win32 API), establishing a direct communication path with the SCM.  
  This registers the handler that receives SCM control messages.  
  (At this time, the system registers whether it wants to receive `Shutdown` or `PreShutdown`.)
  (After this point, SCM control messages no longer flow into `WindowsService.ServiceBase`.)

- `MyServiceBase` invokes the Worker’s `OnStart()`.

- After `OnStart()` completes, the asynchronous task `RunAsync` is started, entering the main loop.

Key points:

- Point 1: All SCM control messages are intercepted by `MyServiceBase.Handler`.  
  As a result, `WindowsService.ServiceBase` no longer receives SCM events.  
  `MyServiceBase` becomes fully responsible for issuing cancellation tokens and managing service shutdown.

- Point 2: Because SCM events are registered directly, the system can receive the `PreShutdown` event.

#### 【STOP】

- The SCM sends `SERVICE_CONTROL_STOP` and sets the state to `STOP_PENDING`.
- `MyServiceBase.Handler` receives the event and sends `CancellationToken = true` to all managed tasks.
- Almost simultaneously, the Worker’s `OnStop` is invoked.
- `MyServiceBase` waits for all managed tasks to complete, then requests the `BackgroundService` (Hosting) to stop.
- The service stops (the executable exits).
- The SCM transitions to `STOPPED`.

If the service does not exit within 30 seconds after `SERVICE_CONTROL_STOP`,  
the SCM forcibly terminates the process.

Key points:

- Point 1: Startup and shutdown are handled by .NET; only SCM control messages are handled independently.  
  The overall architecture still follows the WorkerService model.

- Point 2: The traditionally separate event paths (`OnStart`, `OnStop`, `OnShutdown`, `OnPreShutdown`)  
  are unified into a single interface in `MyServiceBase`, providing a consistent and easy-to-use API for the Worker.

---

## 6.Differences Between .NET 10.0 and .NET Framework 4.8

### — From the Perspective of PreShutdown Implementation —

### 6.1. .NET Framework 4.8 is a “Windows‑only framework”

- `.NET Framework` is Windows‑exclusive and includes **built‑in integration with Windows APIs**, including `ServiceBase`.
- Therefore, simply inheriting from `ServiceBase` allows direct communication with the Windows Service system (`SCM`).
- In `.NET Framework`, modifying `acceptedCommands` (via reflection) allows **enabling the PreShutdown event**, because the underlying implementation is tightly coupled with Windows.

---

### 6.2. .NET 10.0 (.NET Core family) is a “cross‑platform framework”

- `.NET 10.0` treats **Linux / macOS / Windows** as equal targets under a unified abstraction layer.
- As a result, Windows‑specific APIs such as `ServiceBase` are **not included by default**.
- Windows‑only features are provided through the Hosting layer (Generic Host), effectively treating Windows as one of several “virtualized environments”.
- Because of this abstraction, Windows‑specific behaviors come with restrictions.  
  It is likely that enabling Windows‑exclusive features by default would cause cross‑platform build inconsistencies, which is why these capabilities are intentionally limited.

---

### 6.3. The essence of WorkerService

- `BackgroundService` in WorkerService is fundamentally an **interface to the Hosting system**, not to Windows.
- Therefore, WorkerService **does not natively communicate with the SCM**.
- In other words, WorkerService alone **cannot receive or configure SCM events**.

---

### 6.4. How `ServiceBase` is handled in .NET 10.0

- Installing the `Microsoft.Extensions.Hosting.WindowsServices` NuGet package enables WindowsService support.  
  Calling `.UseWindowsService()` causes the system to internally create a `ServiceBase` instance.
- However, this `ServiceBase` is effectively **one‑way**:  
  it can receive control messages from the SCM, but **cannot configure SCM settings**.
- As a result, users **cannot enable PreShutdown**, because the API surface does not expose any way to modify SCM configuration.
- Reflection reveals that `.NET Framework`’s `acceptedCommands` has been replaced with `_acceptedCommands` in `.NET (Core)`;  
  however, **modifying `_acceptedCommands` does not propagate to the SCM**, making it ineffective.

---

### 6.5. Conclusion (current understanding)

- **.NET Framework 4.8**  
  → `ServiceBase` is tightly integrated with Windows and can handle all SCM events, including `PreShutdown`.

- **.NET 10.0 (Core) WorkerService**  
  → Although an internal `ServiceBase` exists, **users cannot modify SCM settings**, making PreShutdown unavailable.

- Therefore, in `.NET 10.0 WorkerService`, the **only practical method** to handle `PreShutdown` is to call the Win32 APIs directly:  
  `RegisterServiceCtrlHandlerEx`, `SetServiceStatus`, and `ChangeServiceConfig2`.

## 7.Considerations (Summary)

- `.NET 10.0 WorkerService` does **not** provide any official API for enabling or handling the `PreShutdown` event.  

- Based on all experiments conducted, the only method that successfully handles `PreShutdown` is to directly call the **Win32 APIs**:  
  `RegisterServiceCtrlHandlerEx`, `SetServiceStatus`, and `ChangeServiceConfig2`.  
  No alternative approach has been found that works reliably.

- Although it is technically possible to inherit from `ServiceBase` within WorkerService,  
  **there is no mechanism to propagate configuration changes to the SCM**,  
  making it impossible to enable or receive the `PreShutdown` event.

- The `ServiceBase` in `.NET Framework 4.8` and the `ServiceBase` used internally by `.NET 10.0 WorkerService`  
  **share the same name but are fundamentally different**.  
  From the SCM’s perspective they behave similarly, but from the user’s perspective they are **not compatible**.

- When focusing solely on SCM event handling,  
  `.NET 10.0 (Core)` offers **less usability** than `.NET Framework 4.8`,  
  because the WorkerService model abstracts away Windows‑specific functionality.

## 8.Example Logs from MyServiceBase Execution

The following tests intentionally avoid stopping the service immediately after receiving a stop request.  
Instead, they measure **how long the service can continue running** after receiving `Shutdown` or `PreShutdown`.

### 8.1. Shutdown Event Test

Verification of behavior during Shutdown, and whether external communication is still possible.  
It is known that **HTTPS is already closed at the moment Shutdown begins**, so only HTTP was tested.

```text
2026/03/07 15:15:38 === MyService2 Main() Started === 
2026/03/07 15:15:38 SV2 Service Started 
2026/03/07 15:16:34 SV2 Shutdown received 
2026/03/07 15:16:34 SV2 Shutdown Process start. 
2026/03/07 15:16:35 SV2 HTTP-POST(plain)[0] = Success 
2026/03/07 15:16:37 SV2 HTTP-POST(plain)[1000] = Success 
2026/03/07 15:16:38 SV2 HTTP-POST(plain)[2000] = Success 
(communication stops here)
Shutdown was received and the connection was cut after 4 seconds.
```

---

### 8.2. PreShutdown Event Test

PreShutdown wait time set to 20 seconds.  
HTTP and HTTPS POST requests were sent simultaneously.

```text
2026/03/07 15:49:39 === MyService Main() Started === 
2026/03/07 15:49:39 Service Started 
2026/03/07 15:50:26 PRESHUTDOWN received 
2026/03/07 15:50:26 PreShutdown Process start. 
2026/03/07 15:50:28 SV1 HTTP-POST(plain)[0] = Success 
2026/03/07 15:50:29 SV1 HTTPS-POST[0ms] = Success
... (all requests succeed during this period) ...
2026/03/07 15:51:06 SV1 HTTP-POST(plain)[19000] = Success 
2026/03/07 15:51:07 SV1 HTTPS-POST[19000ms] = Success 
2026/03/07 15:51:08 SV1 HTTP-POST(plain)[20000] = Success 
(communication stops here)
PreShutdown was received and the connection was cut almost exactly at the configured 20‑second timeout.
```

---

### 8.3. Two Services Running Simultaneously

Testing behavior when two services receive events at different timings.

```text
2026/03/07 15:54:15 PRESHUTDOWN received 
2026/03/07 15:54:15 PreShutdown Process start. 
2026/03/07 15:54:18 SV1 HTTP-POST(plain)[0] = Success 
2026/03/07 15:54:19 SV1 HTTPS-POST[0ms] = Success
... (all requests succeed during this period) ...
2026/03/07 15:55:03 SV1 HTTP-POST(plain)[22000] = Success 
2026/03/07 15:55:03 SV1 HTTPS-POST[22000ms] = Success 
2026/03/07 15:55:04 SV2 Shutdown received   ← SV2 receives Shutdown here
2026/03/07 15:55:05 SV2 Shutdown Process start. 
2026/03/07 15:55:05 SV1 HTTP-POST(plain)[23000] = Success 
2026/03/07 15:55:05 SV1 HTTPS-POST[23000ms] = Success 
2026/03/07 15:55:06 SV2 HTTP-POST(plain)[0] = Success 
2026/03/07 15:55:07 SV1 HTTP-POST(plain)[24000] = Success 
2026/03/07 15:55:07 SV1 HTTPS-POST[24000ms] = Success 
2026/03/07 15:55:08 SV2 HTTP-POST(plain)[1000] = Success 
2026/03/07 15:55:09 SV1 HTTP-POST(plain)[25000] = Success 
2026/03/07 15:55:09 SV1 HTTPS-POST[25000ms] = Success 
(communication stops here)
```

**Important observation:**  
In this example, HTTPS communication during PreShutdown succeeded even beyond the configured 20‑second timeout.  
However, this appears to be because the system aligned the timing with the Shutdown event of the other service.

In other words:

- The system does **not** wait for PreShutdown to finish before issuing Shutdown.  
- Instead, Shutdown may begin **during** PreShutdown, and the system adjusts timing so that both events complete in a coordinated manner.

---

## A1.Appendix - Experimental Attempts and Results (Overview)

### A1.1. Calling .UseWindowsService() and Registering a MyServiceBase (derived from ServiceBase) in Hosting

Under `.NET Framework`, this approach correctly triggers the override methods.  
The expectation was that WorkerService would behave similarly.

**Result: Failure**

- No callbacks from `MyServiceBase` were triggered.  

- After a short time, an exception occurred.  
  It appears that `OnStart` or `StartAsync` was considered to have failed.

- The most reasonable interpretation is:
  
  - The `ServiceBase` instance created internally by WorkerService’s Hosting is a **black box** that the user cannot interact with.
  - When `.UseWindowsService()` is enabled, control messages do **not** propagate to user-defined `ServiceBase` classes.
  - The `OnXXXX` callbacks are either triggered only internally or not triggered at all.

**Impression:**  
It is unclear why this feature exists if it cannot be used meaningfully.

---

### A1.2. Not calling .UseWindowsService() and directly using a class derived from ServiceBase

**Result: Partially successful, but ultimately a failure**

- Default events (`Start` / `Stop` / `Shutdown`) were received.

- However, there is **no way to configure `acceptedCommands` for PreShutdown**.

- Reflection showed that:
  
  - `.NET Framework` uses `acceptedCommands`
  - `.NET (Core)` uses a renamed field `_acceptedCommands`

- Even when `_acceptedCommands` was modified to include `PreShutdown`,  
  **the change was not propagated to the SCM**, resulting in failure.

**Impression:**  
It feels as if the design philosophy is:  
“Default SCM events are allowed, but **do not use PreShutdown**.”

---

### A1.3. Calling .UseWindowsService() Creating a MyServiceBase derived from BackgroundService and attempting to manipulate the internal ServiceBase via reflection

**Result: Failure**

- When `.UseWindowsService()` was enabled, the expectation was that the internal `ServiceBase` instance could be found via reflection.

- However, enumerating the services managed by Hosting revealed that **`ServiceBase` was not visible at all**.

- Since the internal `ServiceBase` clearly functions, the most plausible explanation is:
  
  - It is a **hidden instance** managed outside the normal Hosting service list,  
    or
  - It is a **temporary instance** that disappears after SCM registration.

---

### A14. Final implementation (the `MyServiceBase` published in this document)

**Result: Success**

- `RegisterServiceCtrlHandlerEx` (Win32 API) was used to directly register SCM control events from WorkerService.  
  Since this bypasses .NET entirely, success was expected.
- `SetServiceStatus` and `ChangeServiceConfig2` were used to enable  
  **`SERVICE_ACCEPT_PRESHUTDOWN`** and configure **`PreShutdownTimeout`**.
- Service construction, startup, and shutdown were still handled by WorkerService.  
  Only SCM event handling was implemented independently.
- As a result, it became possible to support  
  **`OnStart` / `OnStop` / `OnShutdown` / `OnPreShutdown`**  
  without breaking the WorkerService model.

**Impression:**  
This approach effectively creates a “side channel” around WorkerService.  
It preserves the WorkerService model, but cannot be used on non‑Windows platforms.  
This raises the question of how meaningful it is to maintain the WorkerService model in such a Windows‑specific scenario.

## S.Author

Author: K.IYO  
This article was written by the author with assistance from Microsoft Copilot.

