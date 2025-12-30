# Track Rat — Bluetooth Connect (Svelte)

This is a minimal Svelte + Vite app that provides a responsive UI with a "Connect" button which attempts to connect to a Bluetooth device using the Web Bluetooth API.

Prereqs
- Node.js (16+ recommended)

Install & run

```bash
npm install
npm run dev
```

Open http://localhost:5173 in a Chromium-based browser (Chrome, Edge) on a secure context (localhost is OK). Click "Connect" to open the Bluetooth device picker.

Notes
- Web Bluetooth requires HTTPS or localhost and is only supported in some browsers (mainly Chromium-based).
- The example uses `acceptAllDevices: true` for demonstration — you should scope `requestDevice` to specific filters/services in production.
- on chrome ubuntu turn on webbluetooth flag
chrome://flags/#enable-web-bluetooth

