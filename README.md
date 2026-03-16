# Neural Network Visualizer

A 3D visualization tool for ONNX models. Drop in your model file and see what's actually happening inside.

## What it does

Takes your ONNX model and renders it as an interactive 3D scene. You can see each layer, watch data flow through the network, and actually understand what's going on instead of just staring at logs.

Built this just as a hobby project.

## Features

- **3D layer visualization** - Conv layers, linear layers, pooling, activations, all rendered in 3D
- **Animated data flow** - Watch activations propagate through the network
- **Interactive** - Click layers to inspect them, rotate/pan the camera
- **Convolutional kernel animation** - See the kernels scan across feature maps
- **No server needed** - Runs entirely in your browser

The animation speed adjusts based on layer type. Conv layers are slower so you can actually see what's happening.

## How to use

1. Open `index.html` in a browser (or visit the GitHub Pages site)
2. Drag and drop your `.onnx` file onto the page
3. Wait a second while it parses
4. Done

### Controls

- Left click + drag to rotate
- Right click + drag to pan
- Scroll to zoom
- Arrow buttons on the right to navigate
- Play button starts the animation
- Click any layer to highlight it

## Architecture

### Overview

The entire application lives in a single `index.html` file (~3,000 lines). There is no build step, no bundler, and no server. All dependencies are loaded from CDN at runtime: React 18, Three.js r128, Babel Standalone (for in-browser JSX transpilation), and protobuf.js 7.2.4.

```
index.html
├── <head>  CDN imports + CSS (lines 1–614)
└── <script type="text/babel">  Entire React app (lines 615–3016)
    ├── Shared geometry cache
    ├── OrbitControls class  (custom camera controller)
    └── NeuralArchitectureExplorer  (root React component)
        ├── State & refs
        ├── File handling
        ├── ONNX parsing
        ├── Layer visualizers
        ├── Three.js scene setup
        ├── Animation loop
        └── JSX UI
```

### Data flow

```
User drops .onnx file
        │
        ▼
handleFile() reads ArrayBuffer
        │
        ▼
parseONNX()  ──success──▶  modelData state (nodes, shapes, params)
        │ failure
        ▼
fallbackParseONNX()  ──────▶  modelData state (nodes inferred from binary)
        │
        ▼
initVisualization()  creates Three.js scene
        │
        ├─▶  createConvLayer()           conv / pool layers
        ├─▶  createLinearLayer()         dense / fully-connected layers
        ├─▶  createActivationLayer()     ReLU, Sigmoid, etc.
        ├─▶  createStructuralLayer()     Add, Mul, Concat
        └─▶  createShapeTransformLayer() Flatten, Reshape, Transpose
        │
        ▼
WebGL render loop  (requestAnimationFrame)
        │
        ▼
animateDataFlow()  highlights active layers, pulses connections
```

### ONNX parsing

**Primary path — protobuf.js:** `onnx.proto` is loaded from the same origin and used to decode the binary file into a `ModelProto` object. This gives full access to node names, operation types, shapes, kernel sizes, and attributes.

**Fallback path — binary pattern matching:** If protobuf decoding fails, `fallbackParseONNX()` scans the raw bytes for UTF-8 strings that match known ONNX op-type names (`Conv`, `Relu`, `BatchNormalization`, etc.). It applies heuristics to discard false positives and caps output at 300 layers for performance.

### 3D rendering

Three.js is used directly (no abstraction layer). The scene is built once after a model is parsed and reused across animation frames.

| Layer type | 3D representation |
|---|---|
| Conv2d / Pool | Triangular prism with stacked feature-map grids and animated kernel planes |
| Linear (Dense) | Two vertical node columns connected by line segments |
| Activation | Ring of small spheres |
| Add / Mul / Concat | Y-shaped merge geometry |
| Flatten / Reshape | Compressed grid showing shape change |

The custom `OrbitControls` class handles mouse/touch input for rotate, pan, and zoom without relying on the Three.js extras bundle.

### Animation system

`animateDataFlow()` runs every frame via `requestAnimationFrame`. It:

- Advances `animationProgress` (0–100%) at a speed scaled by layer type (Conv layers run slower)
- Lights up the currently active layer group by raising mesh opacity
- Pulses connection lines between layers using a travelling colour gradient
- In **tour mode**, drives the camera through five preset motions (orbital rotation, fly-through, zoom wave, top-down sweep, dolly zoom)
- In **exploration mode**, moves the camera continuously along a smooth procedural path

### State management

All state lives in the root `NeuralArchitectureExplorer` component via `useState` / `useRef`. There is no external state library.

| State variable | Purpose |
|---|---|
| `modelData` | Parsed ONNX graph (nodes, shapes, layer count) |
| `isPlaying` | Whether the animation loop is running |
| `animationProgress` | Current playhead position (0–100) |
| `animationSpeed` | User-controlled speed multiplier |
| `selectedLayer` | Index of the layer the user clicked |
| `cameraView` | Active camera preset (default / top / side) |
| `tourMode` | Guided camera tour while animating |
| `explorationMode` | Continuous autonomous camera movement |

## Technical stuff

Uses Three.js for rendering and protobuf.js for parsing ONNX files. The parser tries to decode the protobuf schema first, but if that fails (thanks GitHub Pages CORS issues), there's a fallback that does basic binary pattern matching.

The fallback parser isn't perfect. It looks for operation type strings in the binary data and tries to filter out false positives. Works well enough for most models I've tested.

### Why the fallback parser exists

Originally just used protobuf.js with the external ONNX schema. Worked great locally. Deployed to GitHub Pages and suddenly the external schema URL returns 404. Cool.

So now the protobuf schema is included locally (`onnx.proto`), and there's still a fallback just in case.

## Known issues

- Really large models (500+ layers) get sampled down for performance
- The fallback parser can still detect too many layers on some models, so there's a cap at 300
- Doesn't handle every single ONNX operation type (but covers the common ones)

## Files

- `index.html` - Everything (yeah, it's all one file)
- `onnx.proto` - Local copy of the ONNX protobuf schema

## Models tested

Works with YOLOv8n and various custom models. If your model doesn't parse correctly, check the console. The error messages are actually helpful.

## Development

Just open `index.html`. That's it. No build process, no dependencies to install.

If you want to test locally with file:// protocol, some browsers block that. Python's http.server works fine:

```bash
python -m http.server 8000
```

Then open `http://localhost:8000`

## License

Do whatever you want with it.
