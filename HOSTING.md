# Hosting a study session with public links

Run the study on your laptop and give participants two temporary public links: one for
the 2D app (their laptop browser) and one for the VR page (their Quest browser). This
assumes the repository, `.venv` and requirements are already set up.

## 1. Download cloudflared into the repository root (once)

From the repository root (`FA26_CS567_CrashOverride`):

| System | Command |
| --- | --- |
| WSL / Linux | `curl -L -o cloudflared https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 && chmod +x cloudflared` |
| macOS (Apple silicon) | `curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-darwin-arm64.tgz \| tar xz` |
| Windows PowerShell | `curl.exe -L -o cloudflared.exe https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-windows-amd64.exe` |

Check it works:

```bash
./cloudflared --version          # PowerShell: .\cloudflared.exe --version
```

The file is listed in `.gitignore`, so it is never committed.

## 2. Activate the virtual environment

Do this in **every** terminal you open below, from the repository root:

```bash
source .venv/bin/activate        # PowerShell: .venv\Scripts\Activate.ps1
```

## 3. Start the apps

**Terminal 1: VR page**

```bash
python -m src.backend.main serve
```

It should say it is serving at `http://localhost:8000`.

**Terminal 2: 2D app**

```bash
streamlit run src/frontend/app_2d.py --server.headless true
```

It should say `Local URL: http://localhost:8501`. If it says 8502 instead, an old copy is
still running: stop it first (see "If something goes wrong" below).

## 4. Create the tunnels

**Terminal 3: public VR link**

```bash
./cloudflared tunnel --url http://localhost:8000
```

**Terminal 4: public 2D link**

```bash
./cloudflared tunnel --url http://localhost:8501
```

Each prints a box with an address like `https://some-random-words.trycloudflare.com`.
That is the public link. You get new links every time you start the tunnels, and they stop
working as soon as you press Ctrl+C.

## 5. Test from a different device

Before giving the links to a participant, check them from a device that is **not** on
your Wi-Fi, so you know they are really public. A phone with Wi-Fi turned off (on mobile
data) is easiest.

1. **2D link (terminal 4):** open it on the phone or another laptop. The "Code Review Study"
   page should load. Type `p99` in the sidebar (a test ID, never a real participant), click
   Begin, and complete one trial.
2. **Check the decision saved:** back on your laptop, in the repository root:

    ```bash
    tail -n 1 runtime/decisions.jsonl
    ```

    The last line should show `"participant_id": "p99"`. If the file doesn't exist or has
    no p99 line, the 2D link is reaching an old copy of the app (see below).
3. **VR link (terminal 3):** open it in the Quest browser, enter `p99` and press
   **Start VR session**. The page should load with no certificate warning.
4. **Clean up the test:** remove the p99 lines so they don't mix with real data:

    ```bash
    grep -v '"participant_id": "p99"' runtime/decisions.jsonl > runtime/tmp.jsonl && mv runtime/tmp.jsonl runtime/decisions.jsonl
    ```

When both links work, send them to the participant over the video call (not a public
channel). When the session ends, press Ctrl+C in all four terminals.

## If something goes wrong

**2D app on 8502, or decisions not saving.** An old copy from an earlier session is still
running, often in a closed terminal or VS Code window. Find and stop it:

```bash
ps -eo pid,args | grep -E "streamlit|src.backend.main|cloudflared" | grep -v grep
kill <pid>                       # for each old one
```

PowerShell: `netstat -ano | findstr ":8000 :8501"`, then `taskkill /PID <pid> /F`.

**2D link is blank or stuck on "Connecting".** Restart terminal 2 with:

```bash
streamlit run src/frontend/app_2d.py --server.headless true --server.enableCORS false --server.enableXsrfProtection false
```
