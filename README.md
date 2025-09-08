# ai-smart-wearable
AI Smart Wearable — audio + directional haptic prototype
# AI Smart Wearable — Hear & Feel

**Prototype** that provides audio + direction-aware haptic alerts for visually impaired users.

import { useEffect, useRef, useState } from "react";
import * as cocoSsd from "@tensorflow-models/coco-ssd";
import "@tensorflow/tfjs";

export default function App() {
  const videoRef = useRef(null);
  const canvasRef = useRef(null);

  const [model, setModel] = useState(null);
  const [running, setRunning] = useState(false);
  const [lastPreds, setLastPreds] = useState([]);

  // ---- Helpers ----
  function speak(text, urgent = false) {
    window.speechSynthesis.cancel();
    const u = new SpeechSynthesisUtterance(text);
    u.lang = "en-US";
    u.rate = urgent ? 1.1 : 1;
    u.pitch = urgent ? 1.1 : 1;
    window.speechSynthesis.speak(u);
  }

  function vibrate(dir, danger) {
    if (!navigator.vibrate) return;
    let pattern;
    if (danger === "high") pattern = [400, 100, 400];
    else if (danger === "medium") pattern = [200, 80, 200];
    else pattern = [80];
    if (dir === "left") pattern = [60, 40, ...pattern];
    if (dir === "right") pattern = [40, 60, ...pattern];
    navigator.vibrate(pattern);
  }

  function getDirection(bbox, vw) {
    const cx = bbox[0] + bbox[2] / 2;
    const rel = (cx / vw) - 0.5;
    if (rel < -0.2) return "left";
    if (rel > 0.2) return "right";
    return "ahead";
  }

  function getDangerLevel(obj, vw, vh) {
    const area = (obj.bbox[2] * obj.bbox[3]) / (vw * vh);
    if (["car", "truck", "bus"].includes(obj.class)) return "high";
    if (area > 0.12) return "high";
    if (area > 0.05) return "medium";
    return "low";
  }

  // ---- Camera ----
  async function startCamera() {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({
        video: { facingMode: "environment" },
      });
      videoRef.current.srcObject = stream;
      await videoRef.current.play();
      setRunning(true);
    } catch (e) {
      alert("Camera access denied");
    }
  }

  function stopCamera() {
    const stream = videoRef.current?.srcObject;
    if (stream) stream.getTracks().forEach((t) => t.stop());
    setRunning(false);
  }

  // ---- Detection ----
  async function detectFrame() {
    if (!model || !running) return;
    const preds = await model.detect(videoRef.current);
    const vw = videoRef.current.videoWidth;
    const vh = videoRef.current.videoHeight;
    const ctx = canvasRef.current.getContext("2d");
    canvasRef.current.width = vw;
    canvasRef.current.height = vh;
    ctx.clearRect(0, 0, vw, vh);

    const filtered = preds.filter((p) => p.score > 0.5);
    setLastPreds(filtered);

    filtered.forEach((p) => {
      const [x, y, w, h] = p.bbox;
      ctx.strokeStyle = "yellow";
      ctx.lineWidth = 2;
      ctx.strokeRect(x, y, w, h);

      const dir = getDirection(p.bbox, vw);
      const danger = getDangerLevel(p, vw, vh);

      speak(`${p.class} ${dir}`, danger === "high");
      vibrate(dir, danger);
    });

    requestAnimationFrame(detectFrame);
  }

  // ---- Voice Commands ----
  function setupVoice() {
    const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
    if (!SR) return;
    const recog = new SR();
    recog.continuous = true;
    recog.lang = "en-US";
    recog.onresult = (e) => {
      const phrase = e.results[e.results.length - 1][0].transcript.toLowerCase();
      if (phrase.includes("help")) triggerSOS();
      if (phrase.includes("what")) {
        if (lastPreds.length === 0) speak("No important objects detected");
        else {
          const desc = lastPreds
            .map((p) => `${p.class} ${getDirection(p.bbox, videoRef.current.videoWidth)}`)
            .join(", ");
          speak(desc);
        }
      }
    };
    recog.start();
  }

  // ---- SOS ----
  function triggerSOS() {
    speak("Emergency help activated", true);
    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition((pos) => {
        const msg = `Location: ${pos.coords.latitude}, ${pos.coords.longitude}`;
        console.log(msg);
        alert(msg); // in real app → send to family via API
      });
    }
    navigator.vibrate([500, 200, 500, 200, 700]);
  }

  // ---- Init ----
  useEffect(() => {
    cocoSsd.load().then(setModel);
    setupVoice();
  }, []);

  useEffect(() => {
    if (running) detectFrame();
  }, [running, model]);

  return (
    <div className="min-h-screen bg-black text-white p-4 text-center">
      <h1 className="text-lg font-bold mb-2">AI Smart Glasses — Prototype</h1>

      <div className="relative w-full max-w-md mx-auto">
        <video ref={videoRef} className="w-full rounded-lg" muted playsInline />
        <canvas ref={canvasRef} className="absolute top-0 left-0 w-full h-full" />
      </div>

      <div className="mt-4 flex gap-2 justify-center">
        <button onClick={startCamera} className="bg-green-600 px-4 py-2 rounded-lg">
          ▶ Start
        </button>
        <button onClick={stopCamera} className="bg-gray-700 px-4 py-2 rounded-lg">
          ■ Stop
        </button>
        <button onClick={triggerSOS} className="bg-red-600 px-4 py-2 rounded-lg font-bold">
          🚨 SOS
        </button>
      </div>
    </div>
  );
}






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
