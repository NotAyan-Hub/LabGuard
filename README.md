# LabGuard MVP — School PC monitoring on a local Windows LAN

LabGuard is a read-only monitoring starter for Windows 10/11 PCs on a trusted school Ethernet LAN. One dedicated PC runs a FastAPI server and browser dashboard; each authorized client PC runs a Python agent that sends CPU, RAM, drive-capacity and top-process summaries every 10 seconds.

## MVP includes

- Dashboard inventory with online/offline status, hostname, client IP, CPU/RAM meters, drive capacity, and selected process names/resource usage.
- SQLite storage of reports; dashboard displays the latest report for each hostname.
- Threshold alerts for CPU >= 85%, RAM >= 90%, and a drive >= 90% full.
- PC becomes offline after 35 seconds without a report.
- Bearer-token authentication for report uploads and dashboard API reads.

## Not implemented yet

Windows Event Log collection, SMART disk health, service/startup monitoring, charts/history UI, user accounts, individual per-agent credentials, HTTPS, installer/service deployment, remote actions, and automatic diagnosis. Do not describe these as implemented.

## Requirements

- A dedicated Windows 10/11 server PC and authorized client PCs.
- All devices on the same trusted/private Ethernet LAN.
- Python 3.11 or 3.12 installed on the server and client PCs; check `py --version`.
- Permission from the lab teacher/administrator to install and run the agent and open the server firewall port.

## A. Set up the server PC

1. Copy this project folder to the dedicated lab PC, e.g. `C:\\LabGuard`.
2. Open PowerShell in the project folder and create a virtual environment:

   ```powershell
   py -3 -m venv .venv
   .\.venv\Scripts\python.exe -m pip install --upgrade pip
   .\.venv\Scripts\python.exe -m pip install -r requirements.txt
   ```

3. Create a strong token and save it. Run:

   ```powershell
   .\.venv\Scripts\python.exe scripts\make_token.py
   ```

   Copy the output. Create `.env` from `.env.example` and put the token after `LABGUARD_TOKEN=`. Do not share the token publicly or commit `.env` to source control. Keep the same token available for the agent setup.

4. Find the server's Ethernet IPv4 address:

   ```powershell
   Get-NetIPConfiguration | Select-Object InterfaceAlias,IPv4Address
   ```

   Use the IPv4 address for the Ethernet adapter. Ask the school administrator to create a DHCP reservation or static assignment so the address does not change. The example address `192.168.1.50` is only a placeholder; use the actual school LAN address.

5. Start the server:

   ```powershell
   .\.venv\Scripts\python.exe -m uvicorn server.main:app --host 0.0.0.0 --port 8000
   ```

6. On the server PC, open `http://127.0.0.1:8000`. Click **Set API token**, paste the token, and confirm the dashboard loads. The health endpoint is `http://127.0.0.1:8000/api/health`.

## B. Allow only the trusted school LAN through Windows Firewall

First confirm the Windows network profile is **Private** only if the school administrator approves and the network is genuinely trusted. Do not turn off Windows Firewall.

Example for a lab subnet of `192.168.1.0/24` (replace it with the real subnet): run PowerShell as Administrator on the server PC:

```powershell
New-NetFirewallRule -DisplayName "LabGuard API - School LAN" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 8000 -RemoteAddress 192.168.1.0/24 -Profile Private
```

Do not copy the example subnet blindly. The rule is restricted to the example subnet and Private profile. If the school uses a different subnet, change `-RemoteAddress`. Ask the administrator before making firewall or network changes. Do not port-forward 8000 to the internet.

## C. Install the agent on one test PC

1. Copy the `agent` folder to the test PC. You can copy only `agent.py` and `config.example.json`.
2. Install Python 3.11/3.12 and create a venv. From a PowerShell window in the copied project root (with `agent` folder present), run:

   ```powershell
   py -3 -m venv .venv
   .\.venv\Scripts\python.exe -m pip install --upgrade pip
   .\.venv\Scripts\python.exe -m pip install psutil requests
   Copy-Item agent\config.example.json agent\config.json
   notepad agent\config.json
   ```

   If the client copy contains only the `agent` folder, run the venv commands in that folder and adjust the paths accordingly. Edit `config.json`: set `server_url` to `http://SERVER-IP:8000` and `token` to the exact server token. Keep `interval_seconds` at 10 initially. Do not expose this file; it contains the shared token.

3. Start the agent:

   ```powershell
   .\.venv\Scripts\python.exe agent\agent.py
   ```

   The agent prints whether the report was accepted. The test PC should appear on the dashboard within about 10 seconds. Repeat for the other authorized PCs, using the same server URL and token for this MVP.

## D. Test the system

On the server, open a second PowerShell window:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/api/health
```

To test the protected API, substitute the token locally:

```powershell
$token = "PASTE_YOUR_TOKEN_HERE"
Invoke-RestMethod http://127.0.0.1:8000/api/pcs -Headers @{ Authorization = "Bearer $token" }
```

Test checklist:

- [ ] Server starts without errors.
- [ ] Dashboard token authenticates.
- [ ] One client appears online within 10–20 seconds.
- [ ] CPU, RAM, drives, and process list display.
- [ ] Run a normal CPU-heavy task on the test PC and observe the meter; do not stress shared school PCs without permission.
- [ ] Stop the agent; the PC should become offline after roughly 35 seconds.
- [ ] Restart the agent; the PC should return online.
- [ ] A missing/incorrect token is rejected with HTTP 401.
- [ ] A second PC can report without overwriting the first PC's identity (each PC must have a unique hostname).

Run the basic schema tests from the project root:

```powershell
.\.venv\Scripts\python.exe -m unittest discover -s tests -v
```

## Operational/security notes

- This MVP uses HTTP, so the bearer token and reports are not encrypted on the wire. Use only on a trusted, isolated lab LAN for initial testing. Before production use, configure HTTPS and preferably unique per-agent credentials, rotate tokens, and restrict network access.
- The shared token means any client that knows it can submit reports. It is a bootstrap design, not a full production identity system.
- Run agents visibly under an authorized account. Do not disguise the agent, bypass endpoint protection, or collect unrelated personal information.
- The process list is a point-in-time summary; Windows may deny details for protected processes. CPU values for individual processes may initially be zero because process counters need time to warm up.
- Current storage records each report; long-term retention/cleanup and a history interface should be added before indefinite production use.
- Keep Windows and Python updated. Back up `labguard.db` when the server is stopped.

## Suggested next milestone

Add Windows Event Log error collection (limited to approved System/Application events), drive SMART status where available, a historical chart view, per-agent authentication, HTTPS, and a signed installer/service deployment after the MVP is validated with the school administrator.

## E. Install as background services (no console windows, no Python on lab PCs)

Run all PowerShell commands as Administrator and get permission from the lab administrator first. These scripts were written for Windows 10/11 but have not been tested on real PCs yet; try them on one PC first.

**Teacher/server PC** (after section A is done):

```powershell
powershell -ExecutionPolicy Bypass -File installer\install_server.ps1 -Subnet 192.168.1.0/24
```

This starts the server at every boot and opens port 8000 only to your lab subnet (replace the example).

**Build the agent once** (any PC with Python):

```powershell
powershell -ExecutionPolicy Bypass -File installer\build_agent.ps1
```

This creates `dist\LabGuardAgent.exe`.

**Install on each lab PC** (copy `LabGuardAgent.exe` and `installer\install_agent.ps1` to the PC):

```powershell
powershell -ExecutionPolicy Bypass -File install_agent.ps1 -ServerUrl http://192.168.1.50:8000 -Token "YOUR_TOKEN"
```

The agent runs hidden as SYSTEM, starts at boot, restarts if it crashes, and writes `C:\Program Files\LabGuard\agent.log`. Its token file is readable only by administrators. To remove it, run `installer\uninstall_agent.ps1`.

Reports older than `RETENTION_DAYS` (default 7) are deleted automatically.
