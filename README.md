# Infinite Canvas

A browser-based infinite drawing canvas. Canvases are stored locally in the browser. Shared drawing sessions use direct WebRTC DataChannels with manual offer/answer exchange; no accounts or application backend are required.

## Run it

Serve this folder over HTTP or HTTPS. For local testing, run `py -m http.server 8000` from this folder and open `http://localhost:8000` in a modern browser. For other people to use the app, host it at a public website. WebRTC requires a secure context (HTTPS, or localhost for local testing).

Canvases are autosaved in this browser's local IndexedDB storage. No Google sign-in, online account, or cloud canvas database is used. Private browsing, clearing site data, or resetting the browser can erase local canvases.

## Direct sharing

1. Both people open the same hosted website and keep the app open.
2. The host opens a canvas, clicks **Share**, and selects **Host → Create invitation**.
3. The host privately sends the complete invitation code to one guest.
4. The guest selects **Guest**, pastes the invitation, and clicks **Create response**.
5. The guest privately sends the complete response code to the host.
6. The host pastes the response and clicks **Connect**. Once connected, both people can draw, erase, move or transform strokes, undo their own edits, and update the canvas background together.

The offer and answer are JSON WebRTC session descriptions containing ICE network candidates. They are kept in the page only and are not saved to the canvas database. Send the full codes privately; do not post them publicly. Copy/paste is supported. A short numeric room code is not used because without a signaling service it cannot carry the connection details. QR exchange is not currently included.

The app uses Google's free public STUN service to attempt a direct path between peers. STUN is not a relay. Direct connection can fail behind symmetric or restrictive NAT, some mobile carriers, VPNs, or school/work firewalls. This app does not configure TURN, so those networks will not connect; it cannot guarantee cross-network connectivity. If ICE fails, try another network or a VPN that permits peer-to-peer UDP. No connection is possible while either peer is offline.

WebRTC encrypts the data channel in transit. A connected peer receives the canvas contents and can modify them. The host's canvas is autosaved locally; the guest's shared session is temporary and their previous canvas is restored when they disconnect. The peer connection is one guest per host.

## Controls

- My canvases: this local canvas library is the first screen when the app opens. The Create canvas tile always appears first. Click a canvas to open it; right-click or touch-and-hold a canvas tile for Rename and Delete actions. If a name already exists, choose a preview to replace, keep both (the new one gets a numbered suffix), or cancel to edit the name. Canvas previews use each canvas's background color and show empty canvases without a label; they show as many saved locations as fit in one column. Press Escape or click outside the library to return to an open canvas.
- Share: follow the host/guest offer and answer steps above. Disconnect before switching canvases.
- Draw: drag with mouse, finger, or stylus.
- Paint tools: click the bottom-center tool circle to reveal the fixed tool row. Select a drawing tool, then click it again to open its settings. Pen, Marker, Highlighter, Calligraphy, and Flat Pencil keep their own size, color, and fill settings; Highlighter also has a transparency slider. Shader smudges colors from the canvas. New strokes scale with zoom, becoming smaller as you zoom out and larger as you zoom in; at subpixel sizes the canvas renderer naturally antialiases them. Existing saved strokes retain their prior zoom behavior.
- Drawing from open settings: click the canvas to close settings without drawing, or drag to draw and close settings.
- Lasso: circle a region and reconnect to select its stroke portions. Drag the selection to move it, or drag its top handle to rotate it.
- Ruler: select Ruler to show it in the middle of the screen. Drag its surface to move it; right-click-drag or use two fingers on it to move and rotate it. Drawing pauses where the ruler covers the canvas.
- Zoom and saved views: use the mouse wheel or pinch to zoom from 1e-13% to 1e17%. Each standard mouse-wheel notch changes zoom by 0.01%; pinch zoom remains continuous. The wheel zooms around the pointer, and the canvas can be panned without boundaries. Grid boxes stay visible and adjust their world-space size as you zoom. The canvas starts centered at 0.01% each time it opens. Use Saved locations at the top-left to name and revisit a view's center and zoom; the name/save controls are at the top of the panel, and the list scrolls as it grows. Drag a location's three-dot handle to move it; the row follows the pointer, and nearby locations move to make room.
- Nib angle: Highlighter, Calligraphy, and Flat Pencil have a rotation slider and reset button in their settings. Right-click and drag on the canvas to adjust the selected tool's nib angle.
- Brush cursor: the outline shows the current brush size. The eraser paints with the current canvas background color; it does not remove underlying strokes.
- Dark mode: use the top-right button to switch the canvas between light and dark; black and white canvas colors swap, and the choice is autosaved.
- Lasso outline: dotted marks switch between white and black based on the canvas beneath them.
- Fill: when enabled, closing a stroke fills its enclosed area; leaving it open fills the connected area across the infinite board.
- Pan: hold Space and drag; on touch, use two fingers.
- Undo / redo: always-visible arrows at the bottom-right (redo is disabled until an undo), or Ctrl/Cmd+Z and Ctrl/Cmd+Y.

## How it works

- Strokes are stored as vector point lists in a master `Map`.
- A spatial chunk index maps each 1024 × 1024 world-space chunk to stroke IDs.
- New strokes keep their width in screen pixels and are indexed by their centerlines; saved older strokes retain world-space widths.
- Rendering checks chunks intersecting the viewport and a margin for visible brush widths.
- A stroke crossing chunk boundaries is indexed in every chunk it touches, but its geometry is stored only once.
- Very large strokes are kept outside the chunk index so drawing remains practical at extreme zoom levels.
- The canvas autosaves locally to IndexedDB after edits and camera movement. Only the active host canvas is saved during a peer session; the guest receives a temporary in-memory copy.
- Saved locations, including their view center and zoom level, are stored in the same local IndexedDB document.
