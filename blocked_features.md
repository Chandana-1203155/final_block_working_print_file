# SecurePrint printing restrictions: implemented behavior and limits

SecurePrint's local print enforcement is implemented for Windows. It watches Windows printer queues on the client computer; it is not a network firewall or a USB-device blocker. Read the limitations below before relying on it as the only print control.

## What is monitored and blocked

1. The Electron client starts a persistent PowerShell monitor on Windows. The monitor enumerates printers available through `Get-Printer` and subscribes to native Windows Print Spooler job notifications (`winspool.drv`, `PRINTER_CHANGE_ADD_JOB`). It also scans the queues periodically to catch jobs that notifications may miss.
2. When a job appears on a monitored queue, the watcher tries to pause it while the app decides whether it is authorized.
3. SecurePrint-authorized jobs have a temporary approval tied to an exact, randomly named document copy and, when known, one printer. The approval lasts up to 60 seconds and allows up to three jobs.
4. An unapproved job is deleted from the Windows queue (or cancelled through a slower fallback if it could not be paused). SecurePrint displays a blocked-print warning and queues details locally. If the monitor is not working, the app is intended to show a protection alert rather than silently claim enforcement is active.

This applies to unauthorized jobs submitted through a Windows printer queue that the monitor can enumerate and open. That can include a locally installed USB printer or a Windows-installed network printer: the connection type itself is not what decides authorization.

## USB, network ports, and virtual printers

- **USB printers:** SecurePrint does not disable USB printer devices or USB ports. The installer explicitly restores the Windows `usbprint` and `USBSTOR` services to demand-start and removes a legacy SecurePrint USB printer-class restriction if present. A USB printer's jobs may be monitored when sent through its installed Windows printer queue.
- **Network printers through Windows queues:** A network printer installed as a Windows printer may be monitored like other queues if Windows exposes it to the monitor. Do not assume every shared or remote print-server queue is covered; the current project notes that printers shared over a network are not covered.
- **Direct network printing:** Ports TCP 9100 (RAW), 515 (LPR), and 631 (IPP) are **not blocked by the current enforcement flow**. Helper code can create outbound rules for them, but the enforcement startup calls the cleanup helper instead, and the installer also removes legacy rules with those names. Direct-to-printer network traffic that bypasses a monitored Windows queue may therefore bypass SecurePrint.
- **Virtual/file printers:** Printers matching these ports are skipped: `PORTPROMPT:`, `SHRFAX:`, `nul:`, and `FILE:`. Drivers matching Microsoft Print To PDF, Microsoft XPS Document Writer, or OneNote are also skipped. These virtual or file-output jobs are not blocked by the queue monitor.
- **Printer hiding:** A helper for hiding printers exists in the source but is not called by the normal enforcement startup. Do not expect SecurePrint to hide or disable other printers.

## Client/server availability

- Print monitoring and deletion happen locally on the Windows client; an API connection is not required for the local watcher to detect and cancel an unauthorized queue job.
- Blocked-attempt details are stored in a local queue and sent to the server after a user signs in and the server is reachable. The queue holds at most 200 entries; when full, it drops the oldest entries.
- Login, authorization, quotas, and central record delivery depend on the backend. If the server is unreachable, users cannot rely on the client to perform a fresh online login or confirm centrally authorized printing. Successful print logs are stored in a local outbox for later delivery when possible.
- The installer configures SecurePrint to start at Windows sign-in and a scheduled task to check/restart it periodically. Monitoring does not start before a Windows user signs in. A local administrator can stop the app or remove its scheduled task.

## Requirements and limitations

- The enforcement monitor only runs on Windows and requires the Windows Print Spooler and installed printer queues.
- The NSIS installer is configured per-machine and may ask for administrator elevation. The running app uses the current user's available permissions. Printer access/monitoring can fail if Windows denies the app's required queue-control permissions; check the app's monitor status and protection alerts.
- The app must be installed and running on each computer whose local queues need monitoring. Computers without SecurePrint are not covered.
- SecurePrint is not a Windows service and does not protect printing before a user signs in.
- Local administrators can bypass controls by stopping or modifying the app/task. Direct network protocols, skipped virtual printers, unsupported shared queues, and other paths outside monitored Windows queues may bypass enforcement.
- This is a client-side control, not a guarantee against every way to print. Use Windows/network policy controls as well if your policy requires broader restrictions.

## Safe tests

Use a non-sensitive test document and a test printer. Avoid sending test jobs to a production printer if any page could cause harm or waste.

1. **Check the monitor:** Start SecurePrint on a Windows client. In a development run, look for `Native Windows printer watcher initialized` and `Native ADD_JOB event subscriptions active` in the Electron console. On an installed client, use the blocked-print warning for the test below and ask the administrator to inspect diagnostics if the monitor reports a protection alert. Do not treat a client as protected if monitoring has failed.
2. **Test a physical USB or installed network printer queue:** With SecurePrint monitoring active, send a small direct job from Notepad to a test physical printer. The app should warn and delete the unauthorized queue job. Check the Windows queue to confirm it did not print.
3. **Test authorized printing:** Sign in while connected to the server, then send a small test document through SecurePrint to the same test printer. The authorized job should proceed and be recorded centrally when the server is reachable.
4. **Test virtual-printer exclusions:** Send a harmless test page to Microsoft Print to PDF or another excluded virtual printer. These jobs are intentionally outside SecurePrint's physical-print blocking scope.
5. **Check network-port rules safely:** The current installer/startup removes old SecurePrint rules for TCP 9100, 515, and 631 rather than blocking those ports. Do not send raw jobs to a real printer to test this; inspect Windows Firewall rules or ask the network administrator.
6. **Test offline reporting without relying on a server:** Disconnect a test client from the server network, then make one unauthorized test attempt through a monitored physical queue. The local monitor should still cancel it; the report should remain queued locally until a user signs in while the server is reachable. Do not use a SecurePrint print attempt as a successful authorization test while offline.
