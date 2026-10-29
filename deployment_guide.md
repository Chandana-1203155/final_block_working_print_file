# SecurePrint deployment guide

SecurePrint has a central backend and database, plus a Windows desktop client on each printing computer. Only the server runs Node.js and MySQL. Client computers run the installed SecurePrint application and connect to the server over the network.

## 1. Development computer: install and test

Use a Windows development computer. Download and install Node.js LTS from `https://nodejs.org/` (npm is included) and XAMPP from `https://www.apachefriends.org/` (the project uses its MySQL service and phpMyAdmin). The project does not define a required Node.js version; use an LTS release compatible with its dependencies.

1. Open XAMPP Control Panel and start **MySQL**.
2. Create the project database by importing `database/schema.sql` in phpMyAdmin:
   - Open `http://localhost/phpmyadmin`.
   - Choose **Import**, select the project's `database/schema.sql`, and run the import.
   - Existing databases may still contain the old `failed_login_attempts` and `locked_until` columns. The backend no longer reads or writes them, and its normal schema synchronization does not remove existing columns.
3. In the project root, create or edit `.env`. Use local-only values and replace secret placeholders before using real accounts:

   ```ini
   PORT=3010
   NODE_ENV=development
   DB_HOST=localhost
   DB_PORT=3306
   DB_NAME=monitor_print
   DB_USER=<mysql-user>
   DB_PASS=<mysql-password>
   JWT_SECRET=<long-random-secret>
   JWT_EXPIRES_IN=8h
   APP_BASE_URL=http://localhost:3010
   ```

   The app also reads SMTP settings and optional `SUPER_ADMIN_*` seed settings from `.env`. Do not publish this file or put real secrets in source control. The project creates a default superadmin only when both `SUPER_ADMIN_PASSWORD` and `SUPER_ADMIN_SECRET_CODE` are set and no superadmin exists.
4. Install dependencies and start the backend from the project root:

   ```powershell
   npm install
   npm run backend
   ```

5. In another PowerShell window, check the API:

   ```powershell
   Invoke-RestMethod http://localhost:3010/api/health
   ```

6. To run the desktop client alongside the backend for local development, set `electron/client-config.json` to `"apiBaseUrl": "http://localhost:3010"` and run:

   ```powershell
   npm run dev
   ```

7. Run the tests from the project root when needed:

   ```powershell
   npm test
   ```

## 2. Central server computer: backend and database

The server computer runs the Node.js backend and MySQL. It does not need the SecurePrint desktop client or printer enforcement.

1. Install Node.js LTS/npm and XAMPP with MySQL on the server using their official installers (`https://nodejs.org/` and `https://www.apachefriends.org/`). Copy the project to a stable folder on that computer.
2. Start MySQL in XAMPP and import `database/schema.sql` into phpMyAdmin. The database name in the schema is `monitor_print`.
   Existing databases may retain old account-lockout columns; they are unused by the current backend and do not need to be manually dropped.
3. Create `.env` in the project root on the server. Configure:
   - `PORT=3010` (or choose another port and use it consistently in the firewall and clients).
   - `DB_HOST=localhost`, `DB_PORT=3306`, `DB_NAME=monitor_print`, and the server's MySQL account in `DB_USER` and `DB_PASS`.
   - A unique, long `JWT_SECRET`; do not reuse an example or development value.
   - `APP_BASE_URL` as the server URL for password-reset links.
   - SMTP settings if password reset email is required.
   - `NODE_ENV=development` for an internal HTTP-only setup. In this code, `NODE_ENV=production` redirects HTTP requests to HTTPS, but the backend itself does not configure an HTTPS listener. Use production mode only when a correctly configured HTTPS reverse proxy is in front of it.

   Prefer a dedicated MySQL account with access only to the SecurePrint database. Keep MySQL bound to the server/local machine; client computers need access to the API, not directly to database port 3306.
4. From the project root, install the production dependencies and start the backend:

   ```powershell
   npm install --omit=dev
   npm run backend
   ```

   Keep the process running using the server's approved service/process management method. This repository does not include a backend Windows service installer.
5. Permit inbound TCP traffic to the configured API port (3010 by default) in Windows Firewall, limited to the institution's client network where possible. Do not open MySQL port 3306 to client computers.

   Example for a private LAN; replace the network value with the actual client subnet:

   ```powershell
   New-NetFirewallRule -DisplayName "SecurePrint API" `
     -Direction Inbound -Protocol TCP -LocalPort 3010 `
     -RemoteAddress "<client-subnet>" -Action Allow -Profile Private
   ```

6. On the server, verify the backend and database are running:

   ```powershell
   Invoke-RestMethod http://localhost:3010/api/health
   ```

   Confirm it reports that the SecurePrint API is running. The backend must also show a successful MySQL connection during startup.

## 3. Client computer: install and configure the app

Client computers run only the packaged SecurePrint application. Do not install Node.js, run `backend/server.js`, or install a local SecurePrint database on them.

1. Ensure the client is on a network that can reach the server API, and install the Windows printer driver/printer normally.
2. Before building the installer, set `apiBaseUrl` in `electron/client-config.json` to the server API origin, including protocol and port, but not `/api`:

   ```json
   {
     "apiBaseUrl": "http://<server-ip-or-name>:3010"
   }
   ```

   For an HTTPS deployment, use the HTTPS URL instead. The backend must be reachable at this address from client computers. The Electron app adds `/api` to this base URL.
3. If the server address changes after installation, the client can use the `SECUREPRINT_API_URL` environment variable to override the packaged JSON setting. Set it for the Windows account that runs SecurePrint, then restart the app:

   ```powershell
   [Environment]::SetEnvironmentVariable(
     "SECUREPRINT_API_URL",
     "http://<server-ip-or-name>:3010",
     "User"
   )
   ```

4. Test the URL from the client before installing SecurePrint:

   ```powershell
   Invoke-RestMethod http://<server-ip-or-name>:3010/api/health
   ```

   A connection failure usually means the address, server process, routing, or server firewall needs attention.

## 4. Development computer: build and distribute the Windows installer

1. On the development computer, set `electron/client-config.json` to the server URL intended for these clients. The URL is included in the packaged app unless `SECUREPRINT_API_URL` overrides it at runtime.
2. In the project root, install dependencies and build:

   ```powershell
   npm install
   npm run build:win
   ```

3. Electron Builder writes Windows artifacts to `dist\`. The NSIS installer is named `SecurePrintClient-<version>-Setup.exe`; the version comes from `package.json` (currently `1.0.1`). The build configuration also requests MSI and portable Windows artifacts.
4. Copy the NSIS setup `.exe` to a trusted, access-controlled network share or other approved distribution location. Do not distribute the server's `.env`, source credentials, or development files.
5. On each client, run the setup file. The installer is configured per-machine and can request administrator elevation. Follow the installer prompts, then launch **SecurePrint** from the Start menu or desktop shortcut.
6. For a release update, increment the project version, confirm the server URL, build a new installer, and distribute the new `dist\SecurePrintClient-<version>-Setup.exe`. Install it on client computers using the updated setup. The project does not configure automatic client updates.

## 5. Verify the deployment

1. From a client computer, confirm `http://<server-ip-or-name>:3010/api/health` responds.
2. Launch SecurePrint and sign in with an active account. The application should load its server-backed pages.
3. Submit a small test document through SecurePrint to a test printer. Confirm the job prints and that its record appears in the server's print history/logs.
4. On a client with SecurePrint installed, test a direct print to a physical test printer as described in [blocked_features.md](./blocked_features.md). Use a test document and do not test on a production job.
5. Confirm the client is not running a local backend or MySQL service for SecurePrint.

## 6. Common problems

| Problem | Check |
|---|---|
| Client cannot reach the API | Check the API URL in `electron/client-config.json` or `SECUREPRINT_API_URL`, server address/port, server process, and inbound TCP firewall rule. |
| Backend cannot connect to MySQL | Start MySQL in XAMPP and check `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASS` in the server `.env`. |
| `EADDRINUSE` at backend startup | Another process is using the configured API port. Stop that process or choose a free `PORT`, then update the server firewall and client API URL. |
| API returns an HTTP-to-HTTPS redirect | `NODE_ENV=production` enables an HTTPS redirect in this backend. Use a configured HTTPS reverse proxy, or for a trusted internal test environment use `NODE_ENV=development`; do not expose unencrypted HTTP to untrusted networks. |
| Sign-in works but printing is not monitored | Confirm SecurePrint is running on the client, the Windows Print Spooler is running, the printer is installed, and the monitor reports that it subscribed to printers. See [blocked_features.md](./blocked_features.md). |
| Installer builds but client targets the wrong server | Set `electron/client-config.json` to the intended API URL and rebuild, or set the client-side `SECUREPRINT_API_URL` override. |
