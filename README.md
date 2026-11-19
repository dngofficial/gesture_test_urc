# URC Gesture Control

A camera-based arm gesture interface for a URC rover. The script uses YOLO11 pose estimation, person tracking, arm geometry, and timed holds to produce JSON command intents. CUDA GPU 0 is used by default.

## Gestures

| Command | Gesture |
| --- | --- |
| FOLLOW | One arm raised, the other resting down. Either arm works. |
| STAY | Both arms raised. Displayed internally as STOP. |
| COME | Both arms extended sideways. | (semi functional, but still can detect)
| FETCH (long range) | One arm extended straight sideways, the other down. Either side works. |

Start with both arms down for **0.4 seconds** to arm the detector. Hold FOLLOW, COME, or FETCH for about **0.7 seconds**; STAY needs about **0.35 seconds** and can interrupt another command. Lower both arms again before issuing another movement command. A held gesture emits once.

Only shoulders, elbows, and wrists need to be visible. Hips and legs can be outside the frame. Raised arms allow up to 45 degrees of tilt from vertical and an elbow angle of at least 135 degrees.

## Setup and run

Windows / NVIDIA CUDA setup, tested with Python 3.13:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install torch==2.14.1 torchvision==0.29.1 --index-url https://download.pytorch.org/whl/cu130
.\.venv\Scripts\python.exe -m pip install ultralytics opencv-python numpy lap
.\.venv\Scripts\python.exe urc_gesture_test.py
```

If the project environment is already installed, run only the last command.

The live window shows the current gesture, last emitted command, hold progress, each arm's state, and reasons for rejected poses. A lone person is selected automatically. Click a person's box to select them; target loss or overlapping people may require reselection.

| Control | Action |
| --- | --- |
| Click a person | Select their track |
| `r` | Clear the selection |
| `g` / `v` | Switch gesture / voice mode |
| `q` or Ctrl+C | Quit |

Voice mode is an external recognizer handoff; speech recognition is not included.

## Options

- Long-range vision is enabled by default: 960px tracking plus a 640px target-crop pose pass. The crop is selected when it improves required-joint confidence. Actual recognition range depends on camera detail and joint visibility.
- `--no-long-range` uses lighter 640px tracking without the extra pose pass.
- `--device cpu --no-long-range` runs on CPU.
- `--cam 1` selects another camera; `--half` requests FP16 inference.
- `--up-angle` adjusts raised-arm tilt tolerance; `--up-elbow` adjusts the minimum elbow angle. Larger tilt values and smaller elbow angles are more permissive.
- `--shoulder-frame` uses shoulder-relative directions for a tilted camera. `--require-hips` uses a full-torso reference instead.

Process a recording and save frame results:

```powershell
.\.venv\Scripts\python.exe urc_gesture_test.py --video test.mp4 --log results.jsonl
```

## Rover output and checks

Accepted commands print as JSON. FETCH includes `"command": "FETCH"` and `"range_mode": "long"`. The rover controller must implement the requested travel behavior; this script supplies intents. With ROS 2 sourced and its Python packages available, `--ros-topic /urc/gesture_intent` also publishes JSON through `std_msgs/String`.

Run the deterministic checks without a camera or model inference:

```powershell
.\.venv\Scripts\python.exe urc_gesture_test.py --self-test
```

The checks cover gesture geometry, timing, target identity, crop selection, and command output. They do not measure live-camera accuracy.
