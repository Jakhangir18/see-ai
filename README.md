# SeeAI: object detection in the browser

Real-time object detection on a webcam feed, running entirely in the browser with **YOLOv8n** and **ONNX Runtime Web**. There is no server: the model is downloaded once and inference runs on the device.

## Features

- Live webcam detection of the 80 COCO classes with YOLOv8n
- Inference on the CPU through ONNX Runtime Web's WebAssembly backend
- Boxes and labels drawn over the video on a canvas
- A heads-up display around the feed: FPS, inference time and its recent history, per-class detection counts, a radar view of where objects are in the frame, resolution and model info, and an event log
- Model download progress shown while loading

## How it works

The pipeline is in `src/utils/` and the loop in `src/pages/Index.tsx`.

1. **Load.** `loadModel()` streams `public/yolov8n.onnx` (about 12 MB) with progress reporting and creates an ONNX Runtime session on the `wasm` execution provider.
2. **Capture.** The camera stream (640 x 480 requested) plays in a `<video>` element; each animation frame is copied to an offscreen canvas.
3. **Preprocess.** The frame is resized to 640 x 640 and converted to a normalized `float32` tensor in CHW layout, shape `[1, 3, 640, 640]`.
4. **Infer.** The model returns `[1, 84, 8400]`: 8400 candidate boxes, each with 4 box values and 80 class scores. Candidates below a 0.5 confidence threshold are dropped and boxes are scaled back to the video size.
5. **Non-maximum suppression.** `nms.ts` keeps the highest-scoring box and drops same-class boxes that overlap it with IoU above 0.45.
6. **Draw.** `drawDetections.ts` renders the boxes; the HUD components read the same detection list.

## Getting started

Prerequisites: Node.js 18 or later and npm.

```sh
npm install
npm run dev
```

Open http://localhost:8080 and allow camera access when the browser asks.

Production build:

```sh
npm run build      # output in dist/
npm run preview    # serve the build locally
```

`dist/` is a static site and can be served by any static host.

## Tech stack

| Tool | Purpose |
|---|---|
| [ONNX Runtime Web](https://onnxruntime.ai/) | In-browser model inference (WebAssembly) |
| [YOLOv8n](https://docs.ultralytics.com/models/yolov8/) | Object detection model, exported to ONNX |
| React 18 + TypeScript | UI |
| Vite 5 | Dev server and build |
| Tailwind CSS, shadcn/ui | Styling and UI primitives |
| Framer Motion | HUD animations |
| Three.js | Shader background on the start screen |

## Project structure

```
public/
  yolov8n.onnx            model loaded by the app
  ort-wasm-*.wasm         ONNX Runtime WebAssembly binaries
src/
  pages/Index.tsx         camera, detection loop, layout
  utils/
    detector.ts           model loading, preprocessing, inference, output decoding
    nms.ts                non-maximum suppression
    drawDetections.ts     box and label rendering
    cocoClasses.ts        the 80 COCO class names
  components/hud/         FPS, inference chart, detection breakdown, radar, system metrics, event log, startup sequence
  components/ui/          shadcn/ui components and visual effects
index.html
vite.config.ts
```
