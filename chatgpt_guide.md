# SecurePrint --- Local Run and Server/Client Deployment Guide

This guide explains how to run SecurePrint on one development computer
and how to deploy it so that the **backend and MySQL database run on one
central server**, while each client computer runs only the SecurePrint
Windows application.

> **Important architecture rule:** Do not run the Electron client or the
> print-enforcement service on the central server. Install the
> SecurePrint client only on the Windows computers where printing must
> be monitored. Test the installer on a non-critical computer first
> because it can create Windows firewall rules and integrate with the
> Windows print workflow.

## 1. System architecture

-   **Central server computer:** Node.js/Express backend, MySQL
    database, project `.env`, and server logs.
-   **Client computers:** SecurePrint installer/application (`.exe`) and
    local printer/Windows print monitoring. They do not need the project
    source code, Node.js, XAMPP, or a local SecurePrint database.
-   **Network:** Client computers must be able to reach the server's API
    address on TCP port `3010` (or the port configured in `.env`).
-   **Database:** Only the backend should connect to MySQL. Do not
    expose the MySQL port to client computers.

Example:

``` text
Client PC 1 (SecurePrint.exe) ─┐
Client PC 2 (SecurePrint.exe) ─┼── LAN / HTTPS ──> Server:3010 (Node.js API)
Client PC 3 (SecurePrint.exe) ─┘                         │
                                                         └── MySQL (monitor_print)
```

## 2. Requirements

### Development computer

-   Windows 10/11, 64-bit
-   Node.js LTS and npm
-   The SecurePrint project folder and its `package.json`
-   XAMPP/MySQL for local database testing

### Central server

-   A Windows computer that can remain powered on and connected to the
    network
-   Node.js LTS/npm
-   MySQL or the MySQL service included with XAMPP
-   A stable LAN IP address or a DNS name
-   Permission to configure Windows Firewall
-   For production use, HTTPS configured through a reverse proxy or
    another suitable TLS setup

### Client computer

-   Supported Windows 64-bit computer
-   Local administrator permission for installation and printer/firewall
    integration
-   Network access to the central server
-   Printer and Windows print spooler configured normally

## 3. Run SecurePrint locally on one computer

In this local-development setup, the same computer runs the database,
backend, and Electron client. This is for development/testing only.

### Step 1 --- Extract and open the project

Extract the ZIP to a folder, for example:

``` powershell
C:\Projects\Print_Monitor_blocked_button
```

Open PowerShell in that folder. Confirm that `package.json`, `.env`,
`backend`, `database`, `electron`, and `frontend` exist.

### Step 2 --- Start MySQL

1.  Open the XAMPP Control Panel.
2.  Start **MySQL**. Apache is not required for the Node.js API itself.
3.  Open phpMyAdmin at `http://localhost/phpmyadmin` if available.
4.  Create a database named `monitor_print`.
5.  Import `database/schema.sql` into that database if the tables have
    not already been created.

The `.env` settings must match the MySQL service. The project defaults
are:

``` ini
DB_HOST=localhost
DB_PORT=3306
DB_NAME=monitor_print
DB_USER=root
DB_PASS=
```

If your MySQL installation uses a different port or password, update
these values accordingly. Do not change the port just to match an
example; use the port configured in XAMPP/MySQL.

### Step 3 --- Configure the project `.env`

Create or update `.env` in the project root. Keep real secrets private
and do not share the file or include it in the client installer.

Example for local testing:

``` ini
PORT=3010
NODE_ENV=development

DB_HOST=localhost
DB_PORT=3306
DB_NAME=monitor_print
DB_USER=root
DB_PASS=

JWT_SECRET=REPLACE_WITH_A_LONG_RANDOM_SECRET
JWT_EXPIRES_IN=8h

APP_NAME=SecurePrint
INACTIVITY_TIMEOUT=900000
RESET_TOKEN_EXPIRES_MS=900000

# SMTP values are needed only if the project is using email delivery.
# Configure real credentials before testing email-based password/secret recovery.
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=
SMTP_PASS=
SMTP_FROM=
APP_BASE_URL=http://localhost:3010

# If account lockout has been removed from the code, these variables are not needed.
# MAX_FAILED_LOGINS=5
# LOCKOUT_MINUTES=15

SUPER_ADMIN_USERNAME=superadmin
SUPER_ADMIN_EMAIL=admin@example.com
SUPER_ADMIN_PASSWORD=CHANGE_THIS_INITIAL_PASSWORD
SUPER_ADMIN_SECRET_CODE=CHANGE_THIS_INITIAL_SECRET
```

Use a strong random `JWT_SECRET`. Set a unique initial super-admin
password and secret code. Change them after first login if the
application supports that workflow. Do not use the example credentials
in a real deployment.

**Email note:** If SMTP is not configured, normal login can still work
if the authentication code does not require email delivery. Email-based
forgot-password and secret-recovery flows will not work unless they are
replaced with another recovery method. Do not leave recovery buttons
enabled if those flows cannot send email.

### Step 4 --- Install dependencies

Run in the project root:

``` powershell
npm install
```

### Step 5 --- Start the project

For development, the package script starts the backend and Electron
client together:

``` powershell
npm run dev
```

Alternatively, use two PowerShell windows:

**Window 1 --- backend:**

``` powershell
npm run backend
```

**Window 2 --- Electron client:**

``` powershell
npm start
```

### Step 6 --- Check the backend

Open this URL on the same computer:

``` text
http://localhost:3010/api/health
```

A successful response should indicate that the SecurePrint API is
running. If the backend fails to start, check the terminal for database
connection errors, port conflicts, and invalid `.env` values.

### Step 7 --- Test login and printing safely

1.  Log in with a valid account created through the supported
    administrator workflow.
2.  Verify that the correct role opens the expected dashboard.
3.  Test authorized printing using a test document and a safe test
    printer, such as Microsoft Print to PDF.
4.  Test direct-print blocking only on a computer where you are
    authorized to make printer/firewall changes.
5.  Confirm that print events are logged and that the server database
    receives the expected records.

Printing restrictions may not prevent every possible print path. Do not
treat a "job cancelled" message alone as proof that no physical page was
printed.

## 4. Deploy the backend and database on the central server

The following steps apply when the backend and database are hosted on a
separate server and clients run only the application.

### Step 1 --- Prepare the server

1.  Choose a stable server computer and keep it connected to the same
    trusted network as the client PCs.
2.  Install Node.js LTS.
3.  Install and configure MySQL/XAMPP.
4.  Assign a stable LAN IP address or DNS name. Ask the network
    administrator to reserve an address rather than guessing a gateway
    or IP.
5.  Extract/copy the project source onto the server. The server needs
    the backend code, `package.json`/lockfile, `database/schema.sql`,
    and configuration files. The server does not need to run the
    Electron client.

Example project folder:

``` text
C:\SecurePrint\Print_Monitor_blocked_button
```

### Step 2 --- Create the database

1.  Start MySQL on the server.
2.  Create a database named `monitor_print`.
3.  Import `database/schema.sql` if the schema has not already been
    created.
4.  Create a dedicated MySQL user for SecurePrint with permissions
    limited to this database. Avoid using the MySQL `root` account for a
    real deployment.
5.  Configure `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASS`
    in the server's `.env`.
6.  Keep MySQL bound to localhost where possible. Client computers
    should not connect directly to MySQL.

Example `.env` database values:

``` ini
DB_HOST=localhost
DB_PORT=3306
DB_NAME=monitor_print
DB_USER=secureprint_app
DB_PASS=REPLACE_WITH_DATABASE_PASSWORD
```

If MySQL runs on the same server as the API, `DB_HOST=localhost` is
normally appropriate.

### Step 3 --- Configure the server `.env`

Use a server-side `.env` in the project root. Do not copy this file to
client computers.

``` ini
PORT=3010
NODE_ENV=development

DB_HOST=localhost
DB_PORT=3306
DB_NAME=monitor_print
DB_USER=secureprint_app
DB_PASS=REPLACE_WITH_DATABASE_PASSWORD

JWT_SECRET=REPLACE_WITH_A_LONG_RANDOM_SECRET
JWT_EXPIRES_IN=8h

APP_NAME=SecurePrint
INACTIVITY_TIMEOUT=900000
RESET_TOKEN_EXPIRES_MS=900000

# Configure SMTP only if email-based recovery/notifications are enabled.
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=
SMTP_PASS=
SMTP_FROM=
APP_BASE_URL=http://SERVER-LAN-IP:3010

SUPER_ADMIN_USERNAME=superadmin
SUPER_ADMIN_EMAIL=admin@example.com
SUPER_ADMIN_PASSWORD=CHANGE_THIS_INITIAL_PASSWORD
SUPER_ADMIN_SECRET_CODE=CHANGE_THIS_INITIAL_SECRET
```

Replace `SERVER-LAN-IP` with the server's actual IP address or DNS name.
If email recovery is disabled, `APP_BASE_URL` and SMTP values may not be
needed for normal login, but check that no remaining code path depends
on email.

**HTTPS warning:** This project's server redirects requests to HTTPS
when `NODE_ENV=production`. Do not set `NODE_ENV=production` while
clients still use plain HTTP. Configure HTTPS first, then set
`NODE_ENV=production` and use an `https://` API URL. Plain HTTP is
appropriate only for isolated local testing or a controlled temporary
LAN test; it is not recommended for production credentials or tokens.

### Step 4 --- Install dependencies and start the API

Open PowerShell as appropriate for the installation and navigate to the
project root:

``` powershell
cd C:\SecurePrint\Print_Monitor_blocked_button
npm ci
npm run backend
```

Keep the terminal open during the initial test. Confirm the backend
reports a successful database connection and starts on port `3010`.

For a first test, open this URL on the server:

``` text
http://localhost:3010/api/health
```

Once verified, configure the backend to start automatically after server
restart using an approved Windows service manager or process manager.
Test the service's restart behavior before relying on it. Do not run
multiple backend instances on the same port.

### Step 5 --- Allow API access through Windows Firewall

On the server, open **only the API port** to the trusted client network.
Run PowerShell as Administrator and replace the example subnet with the
actual institution LAN subnet:

``` powershell
New-NetFirewallRule `
  -DisplayName "SecurePrint API 3010" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 3010 `
  -RemoteAddress 192.168.1.0/24 `
  -Action Allow
```

This is an example rule only. Use the correct subnet for your network.
Do **not** open MySQL port `3306` to client computers unless there is a
separate, explicitly justified database-network design.

### Step 6 --- Verify access from another computer

On a client PC, open PowerShell and test the actual server address:

``` powershell
Test-NetConnection SERVER-LAN-IP -Port 3010
```

Then check the API:

``` powershell
Invoke-RestMethod -Uri "http://SERVER-LAN-IP:3010/api/health"
```

Replace `SERVER-LAN-IP` with the server's real address. If this fails,
verify the server process, firewall rule, network profile, and IP
address before installing the client.

## 5. Build the Windows client installer

Build the client installer on the development computer, not on every
client PC.

### Step 1 --- Set the server URL

Edit `electron/client-config.json` before building:

``` json
{
  "apiBaseUrl": "http://SERVER-LAN-IP:3010"
}
```

Replace the example with the server's real LAN address. Use the HTTPS
URL instead when production TLS is configured.

Do not leave `localhost` in the packaged configuration for a separate
server deployment. On a client PC, `localhost` means that client PC, not
the central server.

### Step 2 --- Build the installer

From the project root on the development computer:

``` powershell
npm install
npm run build:win
```

The build output is written to the `dist` folder. Look for the Windows
setup installer with a name similar to:

``` text
dist\SecurePrintClient-1.0.0-Setup.exe
```

The exact filename depends on the version and build output. Confirm the
actual file exists before distributing it.

### Step 3 --- Keep server secrets out of the client

-   Do not copy the server `.env` to a client PC.
-   Do not bundle database credentials or `JWT_SECRET` into the client.
-   The client only needs its server API URL in
    `electron/client-config.json`.
-   Review the installer and packaged files before distribution to
    confirm no secrets were included.

## 6. Install and run the client on each client computer

Each client PC needs the SecurePrint installer and network access to the
server. It does **not** need XAMPP, MySQL, the backend, or the full
source-code project.

1.  Copy the verified installer to the client PC.
2.  Run the installer as Administrator if prompted. The installer is
    configured to require elevation because it integrates with Windows
    printing and firewall settings.
3.  Complete installation.
4.  Ensure the client configuration points to the central server, not
    `localhost`.
5.  Start SecurePrint and log in with an account created by the
    administrator.
6.  Verify that the dashboard loads and that the client can reach the
    API.
7.  Test a document using the approved printing workflow.
8.  Check the server-side audit/print logs to confirm the event was
    recorded.

**Client computer responsibilities:** - Run SecurePrint client. -
Monitor local print activity and enforce the client-side print rules. -
Send authentication and print-log requests to the server. - Queue
supported print logs locally during network outages and retry when
connectivity returns, according to the implementation.

**Server responsibilities:** - Run Node.js/Express API. - Run MySQL and
store users, roles, print logs, reports, and audit records. - Keep
`.env` and secrets private. - Back up the database and monitor server
health. - Do not run the local print-enforcement client on the server.

## 7. Account lockout and email configuration notes

If the account lockout feature has been removed from
`backend/controllers/authController.js`, remove the `MAX_FAILED_LOGINS`
and `LOCKOUT_MINUTES` settings from `.env` as well. The environment
variables alone do not implement or remove behavior; the code and tests
must be updated consistently.

If SMTP/email functionality is removed or not configured: -
Email/password login can still work if the login controller looks up
users by email and verifies the password. - Forgot-password email links,
email OTPs, and email notifications will not work unless replaced with
another mechanism. - Remove or clearly disable frontend recovery options
that depend on sending email. - Keep an administrator-managed account
recovery process.

JWT should remain configured because the current API uses JWTs to
authenticate protected requests. Replace the placeholder `JWT_SECRET`
with a strong random secret on the server.

## 8. Troubleshooting

### Backend cannot connect to MySQL

-   Confirm MySQL is running.
-   Check `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASS`.
-   Confirm the database exists and the schema has been imported.
-   Check the backend terminal for the exact database error.

### Port 3010 is already in use

-   Check whether another SecurePrint backend is already running.
-   Stop the duplicate process or choose a different `PORT` and update
    all client URLs and firewall rules consistently.

### Client says it cannot connect to the server

-   Confirm the API URL in `electron/client-config.json` is the server's
    address, not `localhost`.
-   Confirm the backend is running.
-   Run `Test-NetConnection SERVER-LAN-IP -Port 3010`.
-   Check Windows Firewall and network routing.
-   Confirm HTTP/HTTPS settings match the server configuration.

### Login works locally but not from client PCs

-   Confirm the client can reach `/api/health` on the server.
-   Confirm the client installer was built after setting the correct
    server URL.
-   Check server logs and API responses for authentication or CORS
    errors.

### Email recovery does not work

-   Confirm SMTP is configured with valid provider credentials if email
    recovery is meant to remain enabled.
-   If email has intentionally been removed, use the administrator
    recovery process and disable unsupported email-recovery UI.

### A print job still reaches a printer

-   Print monitoring and cancellation can have timing limitations; job
    notification may occur after the spooler has accepted a job.
-   Some USB, Wi-Fi Direct, shared-printer, or alternate network-print
    paths may behave differently.
-   Validate restrictions on each supported printer/network
    configuration and do not claim universal prevention without testing.

## 9. Deployment checklist

### Server

-   [ ] Node.js installed and backend starts successfully.
-   [ ] MySQL database and schema created.
-   [ ] Dedicated database user configured.
-   [ ] Strong `JWT_SECRET`, database password, and initial
    administrator credentials set.
-   [ ] `.env` stored only on the server and excluded from version
    control/client packages.
-   [ ] API health check succeeds locally.
-   [ ] Firewall allows the API port only from the intended network.
-   [ ] MySQL is not exposed to client PCs.
-   [ ] Database backup and restart procedures tested.
-   [ ] HTTPS configured before using production mode over a network.

### Development/build computer

-   [ ] `electron/client-config.json` points to the correct server API.
-   [ ] Windows installer built successfully.
-   [ ] Packaged files reviewed for secrets.
-   [ ] Installer tested on a non-critical test machine.

### Each client PC

-   [ ] Installer run with required permissions.
-   [ ] Client reaches the server health endpoint.
-   [ ] User can log in.
-   [ ] Authorized printing and logging tested.
-   [ ] Direct-print restrictions tested carefully.
-   [ ] Server records the expected print/audit events.
