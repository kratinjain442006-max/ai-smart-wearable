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
  const [log, setLog] = useState([]);
  const [running, setRunning] = useState(false);
  const [lastPreds, setLastPreds] = useState([]);

  const [settings, setSettings] = useState({
    minScore: 0.5,
    detectionOn: true,
    audioOn: true,
    vibrationOn: true,
  });

  // --- Helpers ---
  function addLog(msg) {
    setLog((prev) => [`${new Date().toLocaleTimeString()} — ${msg}`, ...prev]);
  }

  function speak(text, urgent = false) {
    if (!settings.audioOn) return;
    window.speechSynthesis.cancel();
    const u = new SpeechSynthesisUtterance(text);
    u.lang = "en-US";
    u.rate = urgent ? 1.1 : 1;
    u.pitch = urgent ? 1.1 : 1;
    window.speechSynthesis.speak(u);
  }

  function vibrate(dir, danger) {
    if (!settings.vibrationOn || !navigator.vibrate) return;
    let base;
    if (danger === "high") base = [250, 80, 250];
    else if (danger === "medium") base = [150, 80, 150];
    else base = [80];
    if (dir === "left") base = [60, 40, ...base];
    if (dir === "right") base = [40, 60, ...base];
    navigator.vibrate(base);
  }

  function bboxDirection(bbox, vw) {
    const cx = bbox[0] + bbox[2] / 2;
    const rel = (cx / vw) - 0.5;
    if (rel < -0.22) return "left";
    if (rel > 0.22) return "right";
    return "center";
  }

  function dangerLevel(obj, vw, vh) {
    const area = (obj.bbox[2] * obj.bbox[3]) / (vw * vh);
    if (["car", "truck", "bus", "motorbike", "bicycle"].includes(obj.class))
      return "high";
    if (["person"].includes(obj.class) && area > 0.08) return "medium";
    if (area > 0.12) return "high";
    if (area > 0.05) return "medium";
    return "low";
  }

  function summarize(preds, vw, vh) {
    if (!preds.length) return "No important objects nearby.";
    const parts = preds.map((p) => {
      const dir = bboxDirection(p.bbox, vw);
      return `${p.class} ${dir === "center" ? "ahead" : "to the " + dir}`;
    });
    return parts.join(", ");
  }

  // --- Camera + Detection ---
  async function startCamera() {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({
        video: { facingMode: "environment" },
      });
      videoRef.current.srcObject = stream;
      await videoRef.current.play();
      setRunning(true);
      addLog("Camera started");
    } catch (e) {
      alert("Camera permission denied");
    }
  }

  function stopCamera() {
    const stream = videoRef.current?.srcObject;
    if (stream) stream.getTracks().forEach((t) => t.stop());
    setRunning(false);
  }

  async function detectFrame() {
    if (!model || !running || !settings.detectionOn) return;

    const preds = await model.detect(videoRef.current);
    const filtered = preds.filter((p) => p.score >= settings.minScore);
    setLastPreds(filtered);

    const vw = videoRef.current.videoWidth;
    const vh = videoRef.current.videoHeight;
    const ctx = canvasRef.current.getContext("2d");
    canvasRef.current.width = vw;
    canvasRef.current.height = vh;
    ctx.clearRect(0, 0, vw, vh);

    filtered.forEach((p) => {
      const [x, y, w, h] = p.bbox;
      ctx.strokeStyle = "lime";
      ctx.lineWidth = 3;
      ctx.strokeRect(x, y, w, h);
      ctx.fillStyle = "white";
      ctx.fillText(`${p.class} ${Math.round(p.score * 100)}%`, x + 4, y + 14);

      const dir = bboxDirection(p.bbox, vw);
      const danger = dangerLevel(p, vw, vh);
      speak(`${p.class} ${dir}`, danger === "high");
      vibrate(dir, danger);
    });

    requestAnimationFrame(detectFrame);
  }

  // --- Voice Commands ---
  function setupVoice() {
    const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
    if (!SR) {
      addLog("Voice recognition not supported");
      return;
    }
    const recog = new SR();
    recog.continuous = true;
    recog.lang = "en-US";
    recog.onresult = (e) => {
      const phrase = e.results[e.results.length - 1][0].transcript.toLowerCase();
      addLog("Heard: " + phrase);
      if (phrase.includes("help")) triggerHelp();
      if (phrase.includes("what")) {
        const s = summarize(lastPreds, videoRef.current.videoWidth, videoRef.current.videoHeight);
        speak(s);
      }
    };
    recog.start();
    addLog("Voice commands active");
  }

  // --- SOS Help ---
  function triggerHelp() {
    addLog("🚨 SOS triggered");
    speak("Emergency help activated", true);
    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition((pos) => {
        addLog(`Location: ${pos.coords.latitude}, ${pos.coords.longitude}`);
      });
    }
    navigator.vibrate([400, 100, 400, 100, 600]);
  }

  // --- Init ---
  useEffect(() => {
    cocoSsd.load().then((m) => {
      setModel(m);
      addLog("Model loaded");
    });
    setupVoice();
  }, []);

  useEffect(() => {
    if (running) detectFrame();
  }, [running, model, settings]);

  // --- UI ---
  return (
    <div className="min-h-screen bg-gray-950 text-gray-100 p-4 font-sans">
      <h1 className="text-xl font-bold mb-2">Hear & Feel — Assistive Prototype</h1>

      <div className="relative w-full max-w-lg bg-black rounded-lg overflow-hidden">
        <video ref={videoRef} className="w-full" muted playsInline />
        <canvas ref={canvasRef} className="absolute top-0 left-0 w-full h-full" />
      </div>

      <div className="flex gap-2 mt-3">
        <button
          onClick={startCamera}
          className="px-3 py-2 bg-blue-600 rounded-lg"
        >
          Start
        </button>
        <button
          onClick={stopCamera}
          className="px-3 py-2 bg-gray-700 rounded-lg"
        >
          Stop
        </button>
        <button
          onClick={triggerHelp}
          className="flex-1 px-3 py-2 bg-red-600 rounded-lg font-bold"
        >
          🚨 HELP ME
        </button>
      </div>

      <div className="mt-4 bg-gray-800 p-3 rounded-lg text-sm space-y-2">
        <div>
          <label>
            Min confidence: {Math.round(settings.minScore * 100)}%
            <input
              type="range"
              min="0.2"
              max="0.9"
              step="0.05"
              value={settings.minScore}
              onChange={(e) =>
                setSettings({ ...settings, minScore: parseFloat(e.target.value) })
              }
            />
          </label>
        </div>
        <label>
          <input
            type="checkbox"
            checked={settings.audioOn}
            onChange={(e) =>
              setSettings({ ...settings, audioOn: e.target.checked })
            }
          />{" "}
          Audio Alerts
        </label>
        <label>
          <input
            type="checkbox"
            checked={settings.vibrationOn}
            onChange={(e) =>
              setSettings({ ...settings, vibrationOn: e.target.checked })
            }
          />{" "}
          Vibration
        </label>
      </div>

      <div className="mt-4 bg-gray-900 p-2 rounded-lg h-40 overflow-y-auto text-xs">
        {log.map((l, i) => (
          <div key={i}>{l}</div>
        ))}
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
