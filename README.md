# MyServiceBase for .NET 10 WorkerService

## — An Extended Base Class That Makes Windows Services Easier and Safer —

Search keywords: .NET 10.0 WorkerService, Windows Service, PreShutdown, ServiceBase, shutdown delay

---

## 1. Introduction

This library provides **MyServiceBase**, an extended base class for Windows Services running on **.NET 10.0 WorkerService**.

It offers:

- Functionality equivalent to **.NET Framework 4.8 ServiceBase**
- Full support for the **PreShutdown** event
- A clean interface for handling Start / Stop / Shutdown / PreShutdown
- Safe asynchronous task management
- Clear separation between lifecycle boilerplate and user logic

WorkerService has a major limitation:  
it **cannot receive the PreShutdown event** using the standard WindowsServices package alone.

MyServiceBase solves this limitation and greatly improves **safety**, **development efficiency**, and **maintainability**.

---

## 2. Background

While developing a Windows Service using WorkerService (.NET 10), I discovered that:

- Installing `Microsoft.Extensions.Hosting.WindowsServices` is **not enough**
- WorkerService **cannot receive PreShutdown**
- The internal ServiceBase used by .NET 10 is **not configurable**
- SCM (Service Control Manager) settings cannot be modified by the user

After extensive testing, I concluded that the only reliable way to handle PreShutdown is to call the Win32 APIs directly:

- `RegisterServiceCtrlHandlerEx`
- `SetServiceStatus`
- `ChangeServiceConfig2`

However, this breaks the object‑oriented model and risks inconsistent cleanup.

To solve this, I created **MyServiceBase**, which preserves the WorkerService model while enabling full SCM event handling.

I am publishing this library in the hope that it helps others facing the same issue.

---

## 3. Key Features

### ✔ SCM Event Handling via Override Methods

- `OnStart()` — Service start  
- `OnStop()` — STOP event  
- `OnShutdown()` — Shutdown event  
- `OnPreShutdownAsync()` — PreShutdown event  
  ※ Windows allows **either Shutdown or PreShutdown**, not both.

---

## 4. Practical Features for Real‑World Services

WorkerService requires:

- Starting asynchronous loops after startup  
- Relaying cancellation signals  
- Ensuring safe termination of tasks  

MyServiceBase simplifies all of this:

- Relays CancellationToken to all tasks  
- Tasks terminate when the token becomes true  
- Optional monitoring of asynchronous tasks until they finish  
- Same behavior for Shutdown / PreShutdown  
- Logs are written to user data (text file), not EventLog  
  → Easy to inspect with Notepad  
  → Can be replaced with EventLog if needed

This enables **more reliable and maintainable service development**.

---

## 5. Supported / Tested Environment

- Windows 11 Pro 24H2  
- Visual Studio 2026  
- C# (.NET 10.0)  
- Administrator privileges required for service registration and operation

---

## 6. Included Files (ZIP Packages)

All ZIP files contain real, working samples used during development.

### **MyServiceBase.zip**

- Contains the MyServiceBase class (single C# file).  
- Provided as a source component rather than a class library.

### **Sample1-NetTestWorkerService.zip**

- Buildable project including MyServiceBase.cs.  
- Tests how long external communication remains possible during PreShutdown.  
- Can be used as a template for other services.  
- Includes PowerShell scripts for service registration (requires admin rights).

### **Sample2-Watcher1.zip**

- Worker sample.  
- Monitors SCM status of another service and logs changes.  
- Demonstrates writing logic directly inside RunAsync.  
- Replace `TARGET_SERVICE` with the desired service name.

### **Sample3-SerialCom.zip**

- Worker sample communicating with a microcontroller over a virtual COM port.  
- Demonstrates starting multiple asynchronous tasks in OnStart and returning immediately.

---

## 7. Documentation

Detailed documentation is available under `DOCS/`:

- **User Guide** ([User-Guid.md](DOCS/EN/User-Guide.md))  
- **Technical Note** ([Technical-Note.md](DOCS/EN/Technical-note.md))  
  - Differences between .NET Core and .NET Framework  
  - PreShutdown behavior  
  - Experimental results  
  - Implementation details  
- Source code comments (practical coding advice)

---

## 8. Signature

- Author: K.IYO  
- This library and documentation were designed and written by the author,  
  with assistance from ChatGPT (OpenAI) and Microsoft Copilot.

---

## 9. Copyright / Warranty

- Copyright is **not waived**.
- Free to use and modify for personal or corporate use.
- Redistribution or sale without permission is prohibited.
- No warranty is provided.  
  The author is not responsible for any damages resulting from use of this software.  
  Use at your own risk.

Copyright © 2026.4 K.IYO. All rights reserved.
