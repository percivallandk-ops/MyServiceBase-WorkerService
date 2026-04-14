//==============================================
// Key Points for Using MyServiceBase / Worker
//==============================================

// ■ Logging
// Helper.SetDebugMSGLevel(name, level):
//   Set log level (0=exception, 1=basic, 2=verbose, -1=disabled)
// Helper.Write(message, level):
//   Writes logs to "C:\ProgramData\MyService\MyService.log"
//   Output location can be changed (EventLog output is also possible)

// ■ Task Monitoring (optional)
// registerTask(name, task) / registerTasks(...):
//   When a STOP request is issued, waits up to 8 seconds
//   for registered tasks to finish
//   Unfinished task names are logged (possible zombie state)

// ■ CancellationToken (important)
//   token.IsCancellationRequested becomes true on STOP
//   All tasks must monitor this and terminate promptly
//   Pass the token to subtasks from OnStart / OnPreShutdown

// ■ OnStart Notes
//   OnStart must return immediately (no blocking)
//   Actual work should run in asynchronous tasks
//   Delays may cause SCM to treat startup as failed

// ■ SERVICE_NAME (required)
//   Unique service name; shown in SCM, Task Manager, EventLog

// ■ IsPreShutdown / PreShutdownTimeout
//   IsPreShutdown=true → receive PreShutdown
//   false → receive Shutdown (mutually exclusive)
//   PreShutdownTimeout defines wait time (ms); upper limit depends on system

// ■ Overriding OnXXXX()
//   Override only what you need (others have empty defaults)
//   RunAsync() is typically overridden in Worker
//   If unused, RunAsync() may be removed (e.g., Shutdown-only services)
