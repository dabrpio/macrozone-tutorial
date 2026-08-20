# Expo + React Native + WSL2 + Android Studio — Setup

## 1. Architecture

The goal is to run the React Native / Expo development environment inside **WSL2**, while **Android Studio and the Android Emulator run on Windows**.

```text
                WINDOWS
┌──────────────────────────────────┐
│ Android Studio                   │
│        ↓                         │
│ Android Emulator                 │
│        ↑                         │
│ Windows ADB Server :5037         │
└───────────────▲──────────────────┘
                │
          port forwarding
                │
┌───────────────┴──────────────────┐
│              WSL2                │
│                                  │
│ Expo / React Native              │
│        ↓                         │
│ Linux adb client                 │
│        │                         │
│        └── socat :5554 ──────────┤
└──────────────────────────────────┘
```

The important concept is:

- **WSL2** → code, Node.js, Expo, React Native and `adb` client
- **Windows** → Android Studio, Android Emulator and ADB server
- **Port 5037** → WSL2 `adb` → Windows ADB server
- **Port 5554** → WSL2 → Windows Emulator

---

## 2. Android SDK

The Android SDK is available inside WSL2:

```bash
export ANDROID_HOME="$HOME/Android/Sdk"
```

The emulator itself is **not** running in WSL2. It is managed by Android Studio on Windows.

---

## 3. Connecting WSL2 `adb` to Windows

WSL2 and Windows have different network namespaces, so `localhost` inside WSL2 is not the same as `localhost` on Windows.

First, determine the Windows host IP dynamically:

```bash
export WSL_HOST_IP="$(ip route | awk '/^default/ {print $3; exit}')"
```

Then tell the Linux `adb` client to use the Windows ADB server:

```bash
export ADB_SERVER_SOCKET="tcp:$WSL_HOST_IP:5037"
```

The connection becomes:

```text
WSL2 adb client
      │
      │ TCP :5037
      ▼
Windows ADB server
      │
      ▼
Android Emulator
```

---

## 4. Windows port forwarding for ADB

The Windows ADB server normally listens on `127.0.0.1:5037`, so we expose it to WSL2 using `portproxy`:

```powershell
netsh interface portproxy add v4tov4 `
    listenaddress=0.0.0.0 `
    listenport=5037 `
    connectaddress=127.0.0.1 `
    connectport=5037
```

A Windows Firewall rule allows connections from the WSL2 network:

```powershell
New-NetFirewallRule `
    -DisplayName "ADB WSL2" `
    -Direction Inbound `
    -Action Allow `
    -Protocol TCP `
    -LocalPort 5037 `
    -RemoteAddress <WSL2_SUBNET>/20
```

> > Replace <WSL2_SUBNET> with the WSL2 subnet used by your machine.

---

## 5. Emulator port 5554

ADB connectivity alone is not enough for Expo. The Android Emulator also exposes a console connection, typically on port `5554` for the first emulator.

Because WSL2 has its own `localhost`, we use `socat` as a TCP proxy:

```bash
if ! pgrep -f "socat.*TCP-LISTEN:5554" > /dev/null; then
    socat TCP-LISTEN:5554,bind=127.0.0.1,reuseaddr,fork \
        TCP:$WSL_HOST_IP:5554 >/dev/null 2>&1 &
fi
```

This creates:

```text
WSL2 localhost:5554
        │
        ▼
      socat
        │
        ▼
Windows :5554
        │
        ▼
Android Emulator
```

The `pgrep` check prevents starting multiple `socat` processes when opening new WSL terminals.

---

## 6. Windows forwarding for 5554

The Windows side also needs to forward port `5554`:

```powershell
netsh interface portproxy add v4tov4 `
    listenaddress=0.0.0.0 `
    listenport=5554 `
    connectaddress=127.0.0.1 `
    connectport=5554
```

And allow it through the firewall:

```powershell
New-NetFirewallRule `
    -DisplayName "Android Emulator WSL2 5554" `
    -Direction Inbound `
    -Action Allow `
    -Protocol TCP `
    -LocalPort 5554 `
    -RemoteAddress <WSL2_SUBNET>/20
```

---

## 7. Final `.bashrc`

The relevant configuration in WSL2 is:

```bash
# Android / Expo
export ANDROID_HOME="$HOME/Android/Sdk"

# WSL's adb talks to the Windows adb server
export WSL_HOST_IP="$(ip route | awk '/^default/ {print $3; exit}')"
export ADB_SERVER_SOCKET="tcp:$WSL_HOST_IP:5037"

export PATH="$HOME/bin:$PATH"

# Android Emulator console proxy for Expo
if ! pgrep -f "socat.*TCP-LISTEN:5554" > /dev/null; then
    socat TCP-LISTEN:5554,bind=127.0.0.1,reuseaddr,fork \
        TCP:$WSL_HOST_IP:5554 >/dev/null 2>&1 &
fi
```

---

## 8. Useful checks

Check the Windows host IP:

```bash
echo $WSL_HOST_IP
```

Check that `adb` sees the emulator:

```bash
adb devices
```

Expected:

```text
List of devices attached
emulator-5554    device
```

Check the `socat` process:

```bash
pgrep -af socat
```

Check Windows port forwarding:

```powershell
netsh interface portproxy show all
```

You should see entries for:

```text
5037
5554
```

---

## 9. Mental model

The whole setup can be remembered as:

```text
WSL2                              Windows

Expo / React Native
        │
        ├── adb ───────────────► ADB Server :5037
        │
        └── socat :5554 ───────► Android Emulator :5554
```

**In short:**

> WSL2 runs the development environment, Windows runs Android Studio and the emulator, `5037` connects `adb` to the Windows ADB server, and `5554` provides the additional emulator connection required by Expo.
