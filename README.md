# ai-smart-wearable
AI Smart Wearable — audio + directional haptic prototype
# AI Smart Wearable — Hear & Feel

**Prototype** that provides audio + direction-aware haptic alerts for visually impaired users.

## Demo
Short demo: (add GIF or link to demo video)

## Repo structure
ai-smart-wearable/
├─ detection/
│  ├─ app.py
│  ├─ detector.py
│  ├─ utils.py
│  └─ requirements.txt
├─ esp32/
│  └─ vib_wristband.ino
├─ assets/
│  └─ sample_frame.png
├─ README.md
└─ LICENSE
## Features
- Object detection (YOLOv8)
- Direction-aware vibration (left/right/center)
- Danger-level alerts (short/long vibration + urgent voice)
- Offline TTS (pyttsx3) and BLE haptic commands

## Quick start (local)
1. Clone:
```bash
git clone https://github.com/<your-username>/ai-smart-wearable.git
cd ai-smart-wearable/detection
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\Activate.ps1
pip install -r requirements.txt
python app.py
