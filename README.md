# ⚛️ Ray Tracer: Vue Frontend

This is the Single-Page Application (SPA) built with Vue 3 (or Vue CLI/Vite) that provides the graphical interface for designing optical systems and visualizing the ray tracing simulation results.

It sends component and ray data to the **Flask Backend API** and renders the resulting ray segments using SVG or Canvas.

## 🚀 Deployment (Vercel)

This frontend is designed for deployment on **Vercel** due to its fast static hosting capabilities.

### 🔑 Environment Variable

This application requires one critical environment variable to know where the simulation API lives:

| Variable Name | Description | Example Value |
| :--- | :--- | :--- |
| `VITE_API_URL` (or `VUE_APP_API_URL`) | The public URL of the deployed Flask API. | `https://your-ray-tracer-api.onrender.com` |

---

## 🛠️ Local Development

### 1. Prerequisites

* Node.js (LTS version)
* npm or yarn

### 2. Installation

Navigate to the `frontend/` directory and install dependencies:

```bash
npm install
# OR
yarn install
