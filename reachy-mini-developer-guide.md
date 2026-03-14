<p align="center">
  <img src="images/logo_HP_Electric_Blue_keyline.png" alt="HP" height="60">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
  <img src="images/nvidia-logo-vert.png" alt="NVIDIA" height="60">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
  <img src="images/hf-logo.png" alt="HuggingFace" height="60">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
  <img src="images/pollen-logo.png" alt="Pollen Robotics" height="60">
</p>

<p align="center"><em>Company logos are used for identification purposes only and do not imply endorsement or official partnership unless otherwise stated.</em></p>

# Reachy Mini Developer Guide: 

## What I Wish I'd Known Before Getting Started With The Reachy Mini

*An unofficial field guide for developers building custom apps on the Pollen Robotics Reachy Mini, compiled from hands-on experience building and deploying a voice AI agent.*

*Written and prepared by Curtis Burkhalter, Ph.D., Technical PMM @ HP*

This document covers setup patterns, observed behaviors, and troubleshooting approaches that are not (as of early 2026) well-documented elsewhere. It is written for developers who are comfortable with Linux, SSH, Docker, and Python but are new to the Reachy Mini hardware and SDK.

Where behaviors were observed but not confirmed by official documentation, they are noted as such. Corrections and additions are welcome.

For a working example of a custom Reachy Mini app that uses these patterns, see [curtburk/consent-agent-reachy](https://huggingface.co/spaces/curtburk/consent-agent-reachy) on HuggingFace Spaces.

---

## 1. Understanding the Reachy Mini System Architecture

The Reachy Mini (Wireless version) is a Raspberry Pi 4-based robot running Linux. It has a head with 6 degrees of freedom, two antenna motors, a USB audio device (mic + speaker), and a camera.

The robot is managed by a daemon service called `reachy-mini-daemon`. This daemon:

- Controls all hardware (motors, audio, camera)
- Serves an HTTP API on port 8000 for app lifecycle management
- Runs user apps as Python subprocesses
- Provides a web dashboard at `http://reachy-mini.local`

Your custom app does not run as a standalone process. It runs inside the daemon's subprocess, which has implications for logging, environment variables, and hardware access that are not immediately obvious.

### Key Access Points

```
SSH:        ssh pollen@reachy-mini.local
Web UI:     http://reachy-mini.local
Daemon API: http://reachy-mini.local:8000
Logs:       sudo journalctl -u reachy-mini-daemon -f
```

### Daemon HTTP API

The daemon exposes REST endpoints for app management:

```bash
# List installed apps
curl http://reachy-mini.local:8000/api/apps/list

# Start an app
curl -X POST http://reachy-mini.local:8000/api/apps/start-app/<app_name>

# Stop the current app
curl -X POST http://reachy-mini.local:8000/api/apps/stop-current-app

# Check status
curl http://reachy-mini.local:8000/api/apps/current-app-status
```

---

## 2. App Structure and Deployment

### Required App Structure

A Reachy Mini app is a Python package with a specific structure. The daemon discovers and runs it based on entry points defined in `pyproject.toml`.

```
your_app_name/
├── pyproject.toml
├── README.md
├── your_app_name/
│   ├── __init__.py
│   └── main.py
└── (optional) index.html, style.css
```

The `main.py` must contain a class that extends `ReachyMiniApp`:

```python
from reachy_mini.apps.app import ReachyMiniApp
from reachy_mini.reachy_mini import ReachyMini
import threading

class YourApp(ReachyMiniApp):
    def run(self, reachy_mini: ReachyMini, stop_event: threading.Event):
        # Your app logic here
        while not stop_event.is_set():
            # Do work
            pass
```

The `pyproject.toml` must include the `reachy_mini_apps` entry point group (note: underscore, not dot):

```toml
[project.entry-points.reachy_mini_apps]
your_app_name = "your_app_name.main:YourApp"
```

The `__main__` block in `main.py` is also important because the daemon may run the app as a module:

```python
if __name__ == "__main__":
    app = YourApp()
    try:
        app.wrapped_run()
    except KeyboardInterrupt:
        app.stop()
```

### Deployment via HuggingFace Spaces

The Reachy Mini installs apps from HuggingFace Spaces. Your Space README must include the `reachy_mini_python_app` tag for it to appear in the robot's app store:

```yaml
---
tags:
  - reachy_mini_python_app
---
```

Install from the robot's web dashboard or via the API:

```bash
curl -X POST http://reachy-mini.local:8000/api/apps/install \
  -H "Content-Type: application/json" \
  -d '{"url": "https://huggingface.co/spaces/<user>/<space_name>"}'
```

**Important:** The robot needs internet access to install from HuggingFace. If deploying at venues without internet, install manually via SCP (see next section).

### Manual Deployment via SCP

Apps are installed to the robot's virtual environment at:

```
/venvs/apps_venv/lib/python3.12/site-packages/<your_app_name>/
```

To deploy manually:

```bash
ssh pollen@reachy-mini.local "mkdir -p /venvs/apps_venv/lib/python3.12/site-packages/your_app_name/"
scp your_app_name/main.py pollen@reachy-mini.local:/venvs/apps_venv/lib/python3.12/site-packages/your_app_name/main.py
scp your_app_name/__init__.py pollen@reachy-mini.local:/venvs/apps_venv/lib/python3.12/site-packages/your_app_name/__init__.py
```

After any code update, always clear the Python bytecode cache:

```bash
ssh pollen@reachy-mini.local "rm -rf /venvs/apps_venv/lib/python3.12/site-packages/your_app_name/__pycache__"
```

**Observed behavior:** The daemon caches compiled Python modules. If you update `main.py` without clearing `__pycache__`, the daemon will silently run the old version. This is a common source of confusion when debugging.

---

## 3. Audio: The Most Common Pain Point

### The SDK's Audio Detection Does Not Always Work

The Reachy Mini has a USB audio device ("Reachy Mini Audio") with both a microphone and speaker. The SDK provides `start_recording()`, `get_audio_sample()`, and `push_audio_sample()` methods for audio I/O.

**Observed behavior:** The SDK uses GStreamer's `DeviceMonitor` to find the audio device. In our testing, this consistently failed to detect the audio device when running from the daemon's app subprocess context, logging `"No Reachy Mini Audio Source card found"`. The audio hardware was present and functional; the discovery mechanism did not find it.

We were unable to determine whether this is a permissions issue, a missing environment variable (e.g., `XDG_RUNTIME_DIR`, `DBUS_SESSION_BUS_ADDRESS`), or a GStreamer configuration problem. The daemon subprocess environment lacks several user-session variables that GStreamer may require.

### Workaround: Direct ALSA Recording

The robot's audio device is accessible via ALSA. You can bypass the SDK's audio detection by recording directly with `arecord`:

```bash
# List audio devices
arecord -L | grep reachy

# Record a test clip
arecord -D reachymini_audio_src -f S16_LE -r 16000 -c 2 -d 3 /tmp/test.wav
```

Key details discovered through testing:

- The ALSA device name is `reachymini_audio_src` for the microphone
- The device **requires stereo recording** (`-c 2`). Mono (`-c 1`) fails with "Channels count non available"
- The `hw:CARD=Audio,DEV=0` device is typically locked exclusively by the daemon. Use `reachymini_audio_src` instead, which allows shared access
- There is no PulseAudio or PipeWire running on the robot; it is raw ALSA

In Python, you can use `subprocess.run()` to call `arecord`:

```python
import subprocess

def record_audio(duration_seconds):
    cmd = [
        "arecord", "-D", "reachymini_audio_src",
        "-f", "S16_LE", "-r", "16000", "-c", "2",
        "-d", str(int(duration_seconds)),
        "-t", "raw", "-q", "-"
    ]
    result = subprocess.run(cmd, capture_output=True, timeout=duration_seconds + 5)
    return result.stdout  # Raw PCM bytes, stereo
```

Since this records in stereo, you'll need to convert to mono for most speech processing:

```python
import numpy as np

def stereo_to_mono(pcm_bytes):
    samples = np.frombuffer(pcm_bytes, dtype=np.int16).reshape(-1, 2)
    return samples.mean(axis=1).astype(np.int16).tobytes()
```

### Audio Playback

**Observed behavior:** The SDK's `start_playing()` and `push_audio_sample()` methods work reliably for output, even when audio input detection fails. Playback expects float32 numpy arrays at the output sample rate, which you can query with `reachy_mini.media.get_output_audio_samplerate()`.

---

## 4. Motor Control

### Waking Up the Robot

The robot starts in a "sleep" position (head down, antennas flat). The SDK provides `reachy_mini.wake_up()` to raise the head and play a startup sound.

**Observed behavior:** After a fresh app start, motor commands may not take effect immediately. We observed that calling `goto_target()` right after app startup would execute without errors but produce no physical movement. Starting the app, stopping it, and starting it again reliably enabled motor control on the second launch.

Our workaround was to build a "motor priming" cycle into the launch script: start the app briefly, stop it, wait a few seconds, then start it again. This is not documented behavior and may vary with firmware versions.

### Movement Methods

The SDK provides several methods for controlling the head and antennas:

```python
from reachy_mini.utils import create_head_pose

# Smooth movement to a target pose
head_pose = create_head_pose(yaw=0, pitch=0, roll=0, degrees=True)
reachy_mini.goto_target(head=head_pose, antennas=antennas, duration=0.5)

# Immediate target (no interpolation)
reachy_mini.set_target(head=head_pose, antennas=antennas)

# Antenna-only
reachy_mini.set_target_antenna_joint_positions(antennas)
```

**Observed behavior:** The `antennas` parameter accepts both Python lists and numpy arrays according to the type hints (`Union[ndarray, List[float], None]`). In our testing, we found that passing numpy arrays (`np.array([0.3, -0.3])`) was more reliable than plain Python lists. We observed cases where list values appeared to be silently ignored while numpy arrays produced the expected movement. We did not root-cause this and it may be coincidental or version-dependent.

Antenna values are in radians. The conversation app reference implementation uses `np.deg2rad(15)` (approximately 0.26) as a typical sway amplitude.

---

## 5. Environment Variables and Configuration

### /etc/environment Does Not Propagate to App Subprocesses

**Observed behavior:** Environment variables set in `/etc/environment` on the robot are not available to apps running inside the daemon subprocess. This was confirmed by setting a variable in `/etc/environment`, restarting the daemon, and observing that the app's `os.getenv()` returned `None`.

This means you cannot use environment variables as a reliable configuration mechanism for Reachy Mini apps. If your app needs runtime configuration (e.g., the IP address of an external server), consider:

- Hardcoding it in the app source and updating via `sed` before launch
- Reading from a config file at a known path (e.g., `/tmp/app_config.json`)
- Accepting it as an argument in the app's `run()` method (not currently supported by the daemon's app launcher, to our knowledge)

We chose the `sed` approach for deployment automation because it's simple, debuggable, and doesn't require changes to the daemon.

---

## 6. Networking Across Devices

If your app communicates with an external server (e.g., an AI inference server on another machine), the robot needs to know that server's IP address. This IP changes whenever you connect to a new network.

### mDNS (reachy-mini.local)

The robot advertises itself as `reachy-mini.local` via mDNS. This works on most home and office networks. It may not work on all enterprise or conference networks.

If `reachy-mini.local` doesn't resolve, find the robot's IP from:
- The router's DHCP client list
- The robot's web dashboard (if you can access it from a browser on the same network)
- A subnet scan: `for i in $(seq 1 254); do curl -sf --connect-timeout 0.3 "http://192.168.1.${i}:8000/api/daemon/status" > /dev/null 2>&1 && echo "Found: 192.168.1.${i}"; done`

### Conference / Hotel WiFi

Many conference and hotel WiFi networks enable "client isolation," which prevents devices on the same network from communicating with each other. Symptoms: both devices are connected to WiFi, both have IP addresses on the same subnet, but they cannot reach each other's HTTP endpoints.

Workaround: use a mobile hotspot. Connect both the robot and the server machine to the hotspot. This provides a simple network where devices can see each other.

---

## 7. Logging and Debugging

### Daemon Logs

All app output (stdout, stderr) is captured by the daemon and available via `journalctl`:

```bash
# Live logs (follow mode)
sudo journalctl -u reachy-mini-daemon -f --since 'now'

# Recent logs
sudo journalctl -u reachy-mini-daemon --since '5 min ago'

# Filter for your app
sudo journalctl -u reachy-mini-daemon --since '5 min ago' | grep -i "your_app\|error\|Traceback"
```

**Observed behavior:** The daemon's log output is very noisy with HTTP access logs (every status poll from the web dashboard generates log lines). Filter aggressively to find your app's output.

**Observed behavior:** When an app crashes during import or within the first few seconds, the daemon logs `"App <name> finished"` with no traceback. The RuntimeWarning `"found in sys.modules after import of package"` often precedes a silent crash. To see the actual error, run the app's import manually:

```bash
ssh pollen@reachy-mini.local
sudo /venvs/apps_venv/bin/python3 -c "
from your_app_name.main import YourAppClass
"
```

This will print the actual Python exception.

### Recommendation: Log Everything at Startup

The single most valuable debugging practice is logging every configuration value when your app starts. When something goes wrong remotely, this is the first thing you'll check:

```python
def run(self, reachy_mini, stop_event):
    logger.info("=" * 60)
    logger.info("MY APP STARTING")
    logger.info(f"  Server URL: {SERVER_URL}")
    logger.info(f"  Audio device: {ALSA_DEVICE}")
    logger.info(f"  Python: {sys.version}")
    logger.info("=" * 60)
```

---

## 8. Common Pitfalls

### App crashes silently on startup

The daemon swallows import errors. If your app depends on a package not in the robot's venv, it will crash on import with no visible error in `journalctl`. Test imports manually first:

```bash
ssh pollen@reachy-mini.local "/venvs/apps_venv/bin/python3 -c 'import your_module'"
```

The robot's app venv is at `/venvs/apps_venv/`. Install additional packages there:

```bash
ssh pollen@reachy-mini.local "/venvs/apps_venv/bin/pip install <package>"
```

### Stale __pycache__ after code updates

See Section 2. Always run:

```bash
ssh pollen@reachy-mini.local "rm -rf /venvs/apps_venv/lib/python3.12/site-packages/your_app/__pycache__"
```

### "An app is already running"

Only one app can run at a time. Stop the current app before starting a new one:

```bash
curl -X POST http://reachy-mini.local:8000/api/apps/stop-current-app
sleep 3
curl -X POST http://reachy-mini.local:8000/api/apps/start-app/your_app
```

### Daemon in a bad state

If apps won't start or stop, restart the daemon:

```bash
ssh pollen@reachy-mini.local "sudo systemctl restart reachy-mini-daemon"
```

Wait 30 seconds before attempting to start an app. The daemon needs time to reinitialize hardware connections.

### Robot head doesn't move after starting app

See the motor priming discussion in Section 4. Stop and restart the app once, or build a start/stop/start cycle into your launch script.

---

## 9. Reference: Useful Commands

```bash
# SSH to robot
ssh pollen@reachy-mini.local

# Check daemon status
systemctl status reachy-mini-daemon

# View daemon logs (filtered)
sudo journalctl -u reachy-mini-daemon --since '5 min ago' | grep -v "uvicorn\|GET \|POST "

# List ALSA audio devices
arecord -L | grep reachy

# Test microphone (stereo, 3 seconds)
arecord -D reachymini_audio_src -f S16_LE -r 16000 -c 2 -d 3 /tmp/test.wav

# Check what's in the app venv
/venvs/apps_venv/bin/pip list

# Install a package in the app venv
/venvs/apps_venv/bin/pip install <package>

# Check motor hardware
cat /proc/asound/cards

# Restart daemon
sudo systemctl restart reachy-mini-daemon
```

---

## 10. Things We Don't Know

The following are open questions we encountered but did not resolve:

- **Why does GStreamer's DeviceMonitor fail to find the audio device in the app subprocess context?** The device exists and works via ALSA. The daemon's own GStreamer WebRTC pipeline accesses it fine. This may be a missing environment variable or a session-level permission.

- **Is the motor priming behavior (needing a start/stop/start cycle) a firmware bug or expected behavior?** It is consistent and reproducible but we found no documentation about it.

- **Does the daemon support passing configuration to apps at startup?** We found no mechanism for this beyond hardcoding values in the app source.

- **What is the correct way to access the audio device from a subprocess without bypassing the SDK?** Our ALSA workaround is functional but feels like it should be unnecessary.

---

*This document reflects observations from building and deploying a custom voice AI app on the Reachy Mini in early 2026. SDK and firmware behavior may change with updates. If you found this useful and want to see the full working implementation, visit [curtburk/consent-agent-reachy](https://huggingface.co/spaces/curtburk/consent-agent-reachy) on HuggingFace.*
