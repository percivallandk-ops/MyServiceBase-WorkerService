# How to Use MyServiceBase

1. Environment Setup  
2. Positioning of MyServiceBase (Overview)  
3. Software Development (Coding Worker.cs)  
4. Build  
5. Debugging  
6. Service Deployment (sc.exe)  
Signature  

---

## 1. Environment Setup

**Note:**  
The following steps assume Visual Studio 2026 Community Edition.  
Procedures may differ slightly in other versions.

### 1.1 Installing Visual Studio

Download and install Microsoft Visual Studio from Microsoft’s official website.  
This guide assumes Visual Studio 2026 Community Edition, but other versions may have slightly different menus or workflows.

- You will need to register a Visual Studio account.  
- Even for the free edition, product key and license activation are required.  
- If you are logged into Windows with a Microsoft account, license activation should proceed smoothly.

---

### 1.2 Creating a New Solution and Project (when starting from scratch)

- Launch Visual Studio.  
- Create a **New Project**.  
- In the template search box, enter: **C#, Windows, Service**.  
  This will display templates filtered by  
  **Language: C# / OS: Windows / Project Type: Service**.

  In my environment, this results in two choices:  
  - Worker Service  
  - Windows Service (.NET Framework)

- Since we are creating a WorkerService, select **Worker Service** → [Next].  
- Specify the project name, solution name, and code storage location → [Next].  
- Select **.NET 10.0** as the framework.  
  No other checkboxes are required → [Create].  
  (This automatically generates Program.cs and Worker.cs templates.)

---

### 1.3 Adding Required Packages

To build a Windows Service, you must install an **additional package** for WindowsService support.

- From the menu: **Project → Manage NuGet Packages…**  
  You should already see `Microsoft.Extensions.Hosting` installed.  
  This is added automatically when creating a WorkerService project.

- In the **Browse** tab, search for **WindowsServices**.  
  Install **Microsoft.Extensions.Hosting.WindowsServices**.

This prepares the Generic Host to interface with Windows Services.

---

### 1.4 Adding Required Source Files

Add the following four source files to your project.  
The Worker implementation can be replaced with any of the three provided sample Workers.

- **Program.cs** — Replace the auto‑generated Program.cs.  
- **MyWorker.cs** — Replace the auto‑generated Worker.cs.  
- **MyServiceBase.cs** — Add this file. It is the core middleware.  
- **Helper.cs** — Add this file. It contains support utilities.  

(Optional)  
- **NetHelper.cs** — Required only if you want to run Sample 1.  
  It contains methods referenced by MyWorker.
  
---
## 2. Positioning of MyServiceBase (Overview)

When developing software using this library, it is important to understand the role and behavior of `MyServiceBase`.

- From the perspective of the HostService, MyServiceBase appears to be a Worker.

  - Therefore, when the SCM starts the service, the HostService calls `ExecuteAsync` inside MyServiceBase as if it were a Worker.
  
  - When the service is stopped, the HostService sets the `CancellationToken` to *true*.  
    MyServiceBase exits its waiting loop, returns from `ExecuteAsync`, and requests the HostService to stop the service.  
    Again, it behaves just like a Worker.

- From the perspective of the user‑defined Worker, MyServiceBase appears to be middleware that bridges communication with the SCM.

  - When the SCM issues Start, Stop, Shutdown, or PreShutdown,  
    MyServiceBase invokes the corresponding override methods, regardless of the internal mechanism.
  
  - When the SCM requests a Stop, `OnStop` is called, and all tasks receive a `CancellationToken`.  
    User code should detect this token and terminate tasks as quickly as possible.  
    The SCM enforces a timeout, and if exceeded, the SCM forcibly terminates the service.

- Behavior after MyServiceBase’s `ExecuteAsync` is called at startup:

  - Inside `ExecuteAsync`, MyServiceBase calls `OnStart`.  
    This method can be overridden by the user.
  
  - After returning from `OnStart`, MyServiceBase starts `RunAsync()` asynchronously and waits (`await`).  
    As a result, it remains in a wait state until a `CancellationToken` is received.
  
  - The asynchronously started `RunAsync()` enters a waiting loop and waits for events to occur.

---

## 3. Software Development (Coding Worker.cs)

The user’s primary task is to implement the code in `Worker.cs`.  
No other source files need to be modified.  
(However, if you rename the Worker class in Worker.cs, you must update the class name in Program.cs accordingly.)

---

### 3.1 Creating Overrides in Worker.cs (MyWorker.cs)

(Overview)

1. Configure the **header constants** defined in MyWorker.cs.  
2. Depending on your requirements, implement the **override methods** that correspond to each event.  

That’s all.  
However, since this description alone may not give you a clear coding image, please refer to the three sample Worker.cs files included with this library.  
※ There are important coding considerations described below.

---

#### 3.1.1 Constant Values

- **SERVICE_NAME**:  
  Set the service name.  
  This must be **unique**; otherwise, the service cannot be identified correctly in Task Manager or the Services console.

- **IsPreShutdown**:  
  Set to *true* if you want to receive the PreShutdown event.  
  If *false*, the service will receive the Shutdown event instead.

- **PreShutdownTimeout**:  
  Specifies the **wait time** (in milliseconds) that the SCM should allow during PreShutdown.  
  In principle, the SCM will wait for this duration.  
  This is the only mechanism that allows you to extend the time before Shutdown occurs.  
  However, this value cannot exceed the system’s maximum allowable wait time.  
  Typical limits are around **30–40 seconds (30,000–40,000 ms)**.  
  Since the maximum varies by system environment, measure it on the target system if you need to push the limit.

---

#### 3.1.2 Override Methods

You do not need to implement override methods that you do not use.  
If an override is omitted, MyServiceBase performs the **default behavior** shown below.

Each method receives a `CancellationToken token`.  
This token becomes *true* when the SCM issues a Stop request.

| Override Method       | Trigger Timing     | Purpose                                   | Default Behavior                     |
|-----------------------|--------------------|--------------------------------------------|--------------------------------------|
| OnStart               | On service start   | Initialization, starting subtasks          | Does nothing                         |
| OnStop                | On STOP            | Safe shutdown, resource cleanup            | Does nothing                         |
| OnShutdown            | On Shutdown        | Safe shutdown, resource cleanup            | Does nothing                         |
| OnPreShutdownAsync    | Before Shutdown    | Long-running cleanup, DB flush, etc.       | Does nothing                         |
| RunAsync              | After startup      | Event‑waiting loop                         | 2‑second interval cancel‑wait loop   |

---

### **Important Coding Notes**

- The `CancellationToken token` passed to OnStart and other methods is used to detect Stop requests and terminate tasks safely.  
  Unlike `.NET Framework`’s ServiceBase (which has no parameters), **MyServiceBase intentionally passes this token** so that any subtasks started from OnStart can also receive it.

  The `token` passed to RunAsync is the same token.  
  As long as token monitoring is not blocked, you may place lightweight processing directly inside RunAsync.  
  (See Sample 2.)

- **OnStart must return quickly.**  
  If it does not return promptly, the async wait loop cannot start, and Stop requests may not be detected.  
  A common pattern is to start the actual work as an asynchronous task and return immediately from OnStart.  
  (See Sample 3.)

- `OnPreShutdownAsync` is asynchronous because, while it performs long-running cleanup,  
  MyServiceBase must continue reporting progress to the SCM in parallel.

- `RunAsync()` is typically executed by the Worker.  
  You may override it if needed.

- If your service does not need a waiting loop (e.g., it only handles Shutdown/PreShutdown),  
  you may delete RunAsync entirely from the Worker.  
  This can make the code cleaner.

  (Example of a case where RunAsync does nothing:  
  A service that only responds to Shutdown/PreShutdown.)

---

## 4. Build

There is nothing special required for the build process.  
Build the project **as if it were a normal desktop application**.

The workflow is:

1. Build the project to generate the EXE file.  
2. Register that EXE as a Windows Service using system commands.

(When building an actual desktop application, the build process is the same,  
but the contents of Program.cs would be significantly different.)

---

## 5. Debugging

### 5.1 LOG Output

Debugging Windows Services—especially **during the shutdown sequence**—is extremely difficult.  
For this reason, traditional Debug builds are not very useful.

Instead, the recommended approach is to use **log‑based debugging**, reviewing the log file afterward.  
`Helper.cs` provides methods for writing logs to a file, and logs can be recorded **up until the moment the system shuts down**.

- At the beginning of execution (e.g., at the start of `OnStart`), configure the desired log level:

```cs
Helper.SetDebugMSGLevel(SERVICE_NAME, LogLevel);
```

`LogLevel` values:  
- `0`: Exceptions only  
- `1`: Basic information  
- `2`: Detailed information  
- `-1`: Disable logging  

`SERVICE_NAME`:  
Pass the service name defined in your Worker.  
This allows logs to identify which service produced each message when multiple services write to the same log.

- Write logs anywhere in your code:

```cs
Helper.Write(strings, Level);
```

`strings`: The message to log  
`Level`: The log level (0–2).  
If `Level` is higher than the configured log level, the message is not written.

Example:

```cs
portName = "COM4";
Helper.Write($"{portName} not connected.", 1);
```

This logs a basic‑level message indicating an error condition.

Logs are saved to:

```
C:\ProgramData\MyService\MyService.log
```

The log writer is thread‑safe.  
The output path is hard‑coded; modify Helper.cs if you need a different location.

---

### 5.2 Monitoring Task Completion

When the SCM issues a **STOP** request, MyServiceBase sets the `CancellationToken` to *true*.  
However, if a task fails to terminate for some reason, it may remain running even after the service (parent) has stopped—  
a “zombie task” situation with no visible trace.

To avoid this, MyServiceBase includes a **task monitoring mechanism**.

Users register asynchronous tasks with names so they can be easily identified in logs.

When a STOP request is received:

- MyServiceBase sets the cancellation token to *true*  
- It waits until **all registered tasks** complete or until an **8‑second timeout** occurs  
- If a task does not finish in time, it is **not forcibly killed**, but its name is logged

This helps diagnose which task failed to terminate.

---

#### Registering a single asynchronous task

```cs
registerTask(string name, Task task);
```

`name`: Any identifier (module name, function name, etc.)  
`task`: The task to monitor

---

#### Registering multiple tasks at once

```cs
registerTasks(params (string name, Task task)[] tasks);
```

Example:

```cs
registerTasks(("Task1", task1), ("Task2", task2), ("Task3", task3));
```
---
## 6. Deploying the Service (sc.exe)

Below is the procedure for registering the service using the standard Windows tool **sc.exe**.

**Important:**
- `SERVICE_NAME` must match the name defined in MyWorker.cs.
- Running `sc.exe` requires **administrator privileges**.

The command structure is:

```
sc <command> <ServiceName> <parameters>
```

### Example commands

```cmd
sc stop WorkerService2
sc delete WorkerService2
sc create WorkerService2 binPath= "FULLPATH\WorkerService2.exe" start= auto
sc description WorkerService2 "Description"
sc start WorkerService2
sc query WorkerService2
```

**Note:**  
In `binPath=` and `start=`,  
the **space immediately after "=" is required**.  
This is a specification of the `sc` command.

For detailed command options, refer to Microsoft documentation or other references.

---

### Example output of `sc query WorkerService2`  
(when PreShutdown is successfully enabled)

```cmd
SERVICE_NAME: WorkerService2
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 4  RUNNING
                                (STOPPABLE, NOT_PAUSABLE, ACCEPTS_PRESHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
```

### Key items to verify

- **SERVICE_NAME**:  
  Ensure it matches the intended service name.

- **STATE**:  
  Should be `RUNNING`.

- **ACCEPTS_XXXX flags**:  
  Indicates which SCM events the service accepts.  
  Example:  
  - `ACCEPTS_PRESHUTDOWN` → PreShutdown is enabled and will be received.

## Signature

Author: K.IYO  
This library and its accompanying documentation were designed and written by the author,  
with assistance from ChatGPT (OpenAI) and Microsoft Copilot.  
Copyright and warranty information is provided in the README.

