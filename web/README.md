# InfraNova AI — Interactive Web Client

The interactive web frontend for **InfraNova AI**, built with **React 19** and **Vite 6**. It provides an ultra-responsive, glassmorphic desktop-class interface for cross-spectral translation of Landsat 9 dual-band thermal infrared radiance (Band 10 + Band 11) into true-color optical RGB imagery.

- **Production Live Deployment**: [https://infranovaai.duckdns.org](https://infranovaai.duckdns.org)
- **Interactive OpenAPI Docs**: [https://infranovaai.duckdns.org/docs](https://infranovaai.duckdns.org/docs)

---

## 🌟 Key Features

- **Interactive Comparison Split Slider**: Draggable center-divider with synchronized viewport rendering to compare input thermal radiance against the synthesized optical RGB output in real time.
- **Dual-Mode Thermal Visualization**: Switch between raw thermal radiance input and dynamic false-color colormapping (INFERNO colormap via `/thermal-preview`).
- **One-Click Geographic Presets**: Instant evaluation across pre-loaded Landsat 9 validation scenes:
  - 🌊 **Coastal Urban**: Accra, Ghana (`accra_thermal.tif`)
  - 🌲 **River Basin**: Adilabad, India (`adilabad_thermal.tif`)
  - 🏙️ **Urban Plateau**: Abuja, Nigeria (`abuja_thermal.tif`)
- **Test-Time Augmentation (TTA)**: Toggle 4-fold geometric transformation averaging (original, horizontal flip, vertical flip, combined) for maximum reconstruction fidelity.
- **CLAHE Post-Processing**: Dynamic Contrast-Limited Adaptive Histogram Equalization executed server-side in CIELAB color space ($L^*$ channel only), enhancing visual contrast without altering color chromaticity.
- **Multi-Format Ingestion**: Drag-and-drop or file-picker upload supporting GeoTIFF (`.tif`, `.tiff`), PNG, JPEG, and NumPy `.npy` dual-band rasters.
- **Live Telemetry & Diagnostics**: Real-time display of backend connectivity, inference latency (`X-Inference-Time`), active model identifier (`X-Model`), raster dimensions, and 1-click PNG download.
- **Theming & Design System**: Dark Cosmic (default) and High-Contrast Light modes persisted to `localStorage`, with smooth CSS transitions and bespoke glassmorphism.

---

## 🛠️ Technology Stack

- **UI Framework**: [React 19](https://react.dev/) + [Vite 6](https://vitejs.dev/)
- **Styling Architecture**: Vanilla CSS using custom design tokens and HSL palettes (`src/index.css`)
- **Zero Heavy Component Overhead**: No external bloated UI component kits; 100% custom styling for optimal speed, predictability, and responsiveness
- **Code Quality**: [oxlint](https://oxc.rs/) for high-speed AST linting
- **Client Deployment**: Compiled into static assets (`dist/`) and mounted directly by the FastAPI backend at `/`

---

## 📁 Frontend File Structure

```
web/
├── public/
│   └── samples/             # Bundled Landsat 9 GeoTIFF validation scenes (Accra, Adilabad, Abuja)
├── src/
│   ├── assets/              # Static iconography and graphic assets
│   ├── App.css              # Root application layout constraints
│   ├── App.jsx              # Main dashboard component, state machine, and API handlers
│   ├── index.css            # Complete design system tokens, themes, and glassmorphism styling
│   └── main.jsx             # React DOM entry point
├── index.html               # Single-page application HTML5 shell
├── package.json             # Scripts and dependencies (React 19, Vite 6, oxlint)
└── vite.config.js           # Vite development and build configuration
```

---

## 💻 Local Development

### 1. Prerequisites
- **Node.js**: v18.0.0 or higher
- **npm**: v9.0.0 or higher
- **InfraNova AI Backend**: FastAPI service running (typically at `http://127.0.0.1:8000`)

### 2. Installation

```bash
cd web
npm install
```

### 3. Environment Configuration (Optional)

Create a `.env` file in the `web/` directory if connecting to a decoupled backend:

```env
# Leave empty for same-origin production or set for local development
VITE_API_URL=http://localhost:8000
```

When `VITE_API_URL` is omitted, the frontend automatically routes API calls relatively to the host server.

### 4. Start Development Server

```bash
npm run dev
```

The application will be accessible at:
[http://localhost:5173](http://localhost:5173)

### 5. Code Quality & Linting

```bash
npm run lint
```

Runs `oxlint` across JavaScript and JSX files with strict lint checks.

---

## 📦 Production Build & Container Integration

To generate an optimized production bundle:

```bash
npm run build
```

This compiles minified HTML, CSS, and JS into `web/dist/`.

In Docker production deployments, this build is automated via a multi-stage Docker build:
1. **Stage 1 (Node 20 Alpine)**: Runs `npm install && npm run build`
2. **Stage 2 (Python 3.11 Slim)**: Copies `web/dist/` into `/app/web/dist` and mounts it via FastAPI `StaticFiles(directory="web/dist", html=True)` at `/`

This eliminates Cross-Origin Resource Sharing (CORS) friction and delivers the entire service under a unified port.

---

## 👥 Authors

- **Soham Deshpande**
- **Atharva Lingayat**
- **Lokesh Gakhre**
- **Sanchit Wakde**
