---
name: interactive-product-demo-website
description: Build high-craft product marketing websites that showcase apps, tools, or SaaS products through interactive, frame-by-frame snapshot choreography and live DOM animations instead of video playback (inspired by backdrop-app.com). Use whenever the user asks to create a product landing page, interactive product demo website, non-video app showcase, scroll-driven product tour, or simulated UI stage where product screenshots and UI layers animate like real product usage.
license: MIT
compatibility: Designed for AI coding and web-generation agents supporting HTML5, CSS3 hardware-accelerated transforms, and vanilla ES6+ JavaScript or React/Next.js without external video dependencies.
metadata:
  author: ai-skills
  version: "1.1.0"
  spec: "https://agentskills.io/specification"
  reference-site: "https://backdrop-app.com/"
---

# Interactive Product Demo Website

Build product marketing websites where the product demo is a live, interactive stage constructed from stitched UI snapshots, layered DOM chrome, spring-physics animations, and choreographed cursor and keyboard sequences rather than embedded video files.

## When to Use

- The user wants a product, desktop app, CLI tool, browser extension, mobile app, or SaaS landing page with an embedded live demo instead of an MP4 or screen recording.
- The user references `backdrop-app.com` or asks for a scroll-pinned device stage that zooms into an interactive desktop, browser, or app workspace.
- The user wants to stitch static product screenshots, UI slices, or component frames together and animate transitions, window arrangements, drag-and-drop gestures, or hotkey workflows as if the real product is running.
- The user wants a guided multi-chapter product tour (`01`, `02`, `03`) where visitors can either watch auto-choreographed usage or click pills and press keyboard shortcuts to test features directly.

## Step-by-Step Instructions

### 1. Deconstruct the Product into Snapshot Layers and States

Avoid treating the product demo as a flat screenshot or video stream. Break the product down into independent visual layers that can move, scale, and reorder on a stage:

- **Stage Frame (Device or Viewport Shell):** The outer container representing a MacBook screen, browser frame, mobile bezel, or desktop wallpaper (`16:10` or `16:9` aspect ratio) with system elements like a menu bar, camera notch, or dock.
- **Snapshot Tiles (Product Windows or Panels):** Individual crisp image snapshots (`.webp` or `.png` at `2x` retina density) or hybrid HTML/image cards for each movable product surface (such as an editor window, terminal panel, sidebar, floating HUD, or modal). Wrap each snapshot inside an HTML window frame with live UI chrome (traffic-light dots, title bar, active border glow, and status badges) so text and controls stay razor-sharp at any scale.
- **Interactive Overlays (HUDs, Shelves, Popovers, Cursor):** Floating UI elements that react to state changes, plus an autonomous simulated cursor (`.demo-cursor`) and keypress HUD badge (`Ctrl+Opt+1` or `Cmd+K`) that visually bridge user actions and product responses.
- **Discrete Product States (Chapters):** Define 3 to 4 signature workflows (for example, `01 Switch Contexts`, `02 Arrange Layouts`, `03 Drag Gesture`) where each workflow maps to deterministic `[x, y, width, height, scale, opacity, zIndex]` target rectangles for every snapshot tile.

If the user has not provided raw screenshots for each panel, generate high-resolution synthetic UI snapshots (via SVG, Canvas, or semantic HTML sub-windows with realistic domain data) so every window looks like an authentic capture of the product in action.

### 2. Build the Scroll-Pinned Stage Dive (Hero to Interactive Demo)

Create a seamless transition from the hero headline into the full-screen interactive demo stage using a scroll-pinned container:

- Wrap the hero and stage in a tall scroll track (`.hero-dive`) with `height: calc(100svh * 4)` (or up to `800svh` for multi-phase scroll intros) and `margin-bottom: -100svh`.
- Inside `.hero-dive`, place `<section class="hero-sticky">` with `position: sticky; top: 0; height: 100svh; overflow: hidden;`.
- Compute a normalized scroll progress custom property `--ptn` (`0.0` to `1.0`) in a `requestAnimationFrame`-throttled scroll listener and write it to the stage container.
- Use `--ptn` to interpolate the device mockup from a framed hero perspective (`scale(0.76) translateY(36px)`) to a centered interactive workspace (`scale(1) translateY(0)`), fading out hero marketing overlays as the interactive screen overlay (`[data-interactive="true"]`) activates.
- Provide a direct chapter navigation bar (`01`, `02`, `03`) so users can jump straight to any interactive demo chapter.

```css
.hero-dive {
  position: relative;
  height: calc(100svh * 4);
  margin-bottom: -100svh;
}

.hero-sticky {
  position: sticky;
  top: 0;
  height: 100svh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.hero-stage {
  --scale: calc(0.76 + (0.24 * var(--ptn, 0)));
  --ty: calc(36px * (1 - var(--ptn, 0)));
  transform: translate3d(0, var(--ty), 0) scale(var(--scale));
  will-change: transform;
}
```

### 3. Implement the Snapshot Compositor and Spring Physics Engine

Drive all window and panel movements through a coordinate compositor backed by spring physics rather than linear CSS keyframes so every transition feels like a native OS or live application:

- **Normalized Coordinate Space:** Define the stage's internal coordinate system in percentages (`0..100%` of `.screen-overlay`) or a fixed logical resolution (`1120 x 700`) scaled via CSS `transform: scale(var(--stage-scale))` so bounding-box math stays resolution-independent.
- **Bounding-Box Layout Map:** Store target rectangles `[x, y, w, h]` for every snapshot window across every product state and layout preset.
- **Minimize and Offscreen Parking:** When a snapshot window is inactive or minimized in a given state, animate its target rect toward a dock icon coordinate (`[dockX, dockY, 6, 6]`) with `scale(0.12)` and `opacity: 0`, rather than toggling `display: none`. Never unmount window DOM nodes during state transitions.
- **Spring Physics (`motion.js`):** Interpolate `x`, `y`, `width`, `height`, and `scale` using a damped spring solver (`stiffness: 210`, `damping: 24`, `mass: 1`) in a single `requestAnimationFrame` loop, or apply GPU-accelerated `translate3d(x, y, 0) scale(sx, sy)` with `cubic-bezier(0.22, 1, 0.36, 1)`.

```javascript
export class SpringAnimator {
  constructor({ stiffness = 210, damping = 24, mass = 1 } = {}) {
    this.stiffness = stiffness;
    this.damping = damping;
    this.mass = mass;
    this.items = new Map();
    this.rafId = null;
  }

  setTarget(id, targetProps, onUpdate) {
    let state = this.items.get(id);
    if (!state) {
      state = {
        current: { ...targetProps },
        velocity: Object.fromEntries(Object.keys(targetProps).map(k => [k, 0])),
        target: { ...targetProps },
        onUpdate
      };
      this.items.set(id, state);
      onUpdate(state.current);
      return;
    }
    state.target = { ...state.target, ...targetProps };
    state.onUpdate = onUpdate;
    this.startLoop();
  }

  startLoop() {
    if (this.rafId) return;
    let lastTime = performance.now();
    const step = (now) => {
      const dt = Math.min((now - lastTime) / 1000, 0.032);
      lastTime = now;
      let activeCount = 0;

      for (const state of this.items.values()) {
        let moving = false;
        for (const key of Object.keys(state.target)) {
          const x = state.current[key];
          const target = state.target[key];
          const v = state.velocity[key] || 0;
          const force = -this.stiffness * (x - target) - this.damping * v;
          const nextV = v + (force / this.mass) * dt;
          const nextX = x + nextV * dt;

          if (Math.abs(nextV) > 0.05 || Math.abs(target - nextX) > 0.08) {
            moving = true;
            state.velocity[key] = nextV;
            state.current[key] = nextX;
          } else {
            state.velocity[key] = 0;
            state.current[key] = target;
          }
        }
        state.onUpdate(state.current);
        if (moving) activeCount++;
      }

      this.rafId = activeCount > 0 ? requestAnimationFrame(step) : null;
    };
    this.rafId = requestAnimationFrame(step);
  }
}
```

### 4. Choreograph Autonomous Cursor, Drag, and Keyboard Sequences

For features that involve gestures (such as dragging a window to a drop zone, scrubbing a timeline, or invoking a command palette), build a deterministic timeline sequencer instead of playing a video:

- **Simulated Cursor Element:** Render an SVG pointer element (`.demo-cursor`) positioned absolutely inside `.screen-overlay` with `pointer-events: none` and `z-index: 999`.
- **Step-Based State Machine:** Represent each choreography sequence as an array of timestamped keyframes updating `data-step="1..N"` on the stage container:
  1. Move `.demo-cursor` from rest position to the target window's title bar.
  2. Trigger a press state (`data-cursor="grabbing"`, subtle window lift `scale(0.95)`, and elevated shadow).
  3. Interpolate both `.demo-cursor` and the grabbed snapshot window toward the target drop zone (such as a top notch shelf, sidebar slot, or split zone), expanding the drop zone (`data-drop-active="true"`) when distance falls below a threshold.
  4. Release the window (`data-cursor="default"`), snap the window and sibling windows into their new layout coordinates via the spring engine, and display a confirmation state.
- **Auto-Play, Pause on Interaction, and Replay:** Auto-start a chapter's choreography when it scrolls into view (via `IntersectionObserver`). Immediately pause auto-stepping if the user clicks a control pill or presses a hotkey, and always include a **Replay** button (`data-action="replay"`) for gesture chapters.

### 5. Build Guided Chapters with Dual Control (Click + Hotkeys)

Structure the demo narrative into numbered chapters (`01`, `02`, `03`) positioned alongside or below the pinned stage:

- **Chapter Header and Counter:** Display clear section markers (`01 / Projects`, `02 / Layouts (1 of 5)`, `03 / Gesture`) with concise two-sentence copy explaining what happens on screen.
- **Interactive Control Pills:** Provide pill buttons for every state within the chapter (for example, `1 School`, `2 Development`, `3 Personal` or layout presets `Main and stack`, `Side by side`, `Grid`, `Focus`) plus a `Next` progression button.
- **Live Keyboard Shortcuts:** Attach a global `keydown` listener (ignoring inputs and textareas) so pressing `1`, `2`, `3`, or `Space` switches the stage state identically to clicking the pills, highlighting the corresponding keycap badge in the UI.
- **DOM Data-Attribute State:** Store active states on the root stage element (`data-chapter="projects"`, `data-project="dev"`, `data-layout="main-stack"`, `data-step="2"`) so CSS rules and JS controllers stay synchronized from a single source of truth.

### 6. Verify Visual Craft and Interactive Fidelity

Before delivering the website, verify the implementation against these checks:

- Confirm zero `<video>` tags are used for the interactive product demo chapters (an optional hero device boot animation is permissible only if it transitions to the live DOM stage upon completion or scroll).
- Test every pill button, `Next` button, `Replay` trigger, and keyboard shortcut (`1`, `2`, `3`) to confirm smooth spring transitions between snapshot states with no layout jumps or flickering images.
- Verify that all snapshot images preload cleanly (`new Image()` or `<link rel="preload">`) so switching states never flashes a blank frame.

## Examples of Inputs and Outputs

### Example 1: Mac Window Manager or Desktop Productivity App

- **User Input:** "Create a landing page for my Mac window manager app with an interactive non-video demo like backdrop-app.com. Show 3 project workspaces (Design, Engineering, Writing), 4 tiling layouts, and a drag-to-menu-bar drop gesture."
- **Expected Output:** A single-page application or HTML/CSS/JS bundle featuring:
  1. A `.hero-dive` scroll track (`400svh`) that zooms into a MacBook `.screen-overlay`.
  2. Six persistent `.demo-window` DOM cards containing crisp UI snapshots (Figma, VS Code, Terminal, Safari, Notes, Preview) mapped to owner workspaces.
  3. Chapter `01` switcher with pill buttons (`1 Design`, `2 Engineering`, `3 Writing`) and keyboard listener (`1`, `2`, `3`) that spring-animate active windows into place while minimizing inactive windows to dock coordinates.
  4. Chapter `02` layout selector (`Main & Stack`, `Side by Side`, `Grid`, `Focus`) that recalculates bounding boxes `[x, y, w, h]` dynamically.
  5. Chapter `03` autonomous `.demo-cursor` sequence that grabs a window title bar, drags it to the top bar shelf, drops it into a project slot, and provides a `Replay` button.

### Example 2: SaaS Web App or AI Developer Tool

- **User Input:** "Build a product showcase website for our AI API testing tool without using any video. Stitch snapshots of the request builder, live trace waterfall, and schema diff viewer so users can click through workflows."
- **Expected Output:** An interactive browser-framed stage where clicking workflow pills (`01 Compose Request`, `02 Inspect Trace`, `03 Diff Schema`) or pressing hotkeys animates the panel snapshots, highlights active request nodes, and runs a simulated cursor click-and-scrub across the trace timeline.

## Common Edge Cases and Gotchas

- **Never unmount or `display: none` inactive snapshot windows:** Hiding inactive windows with `display: none` breaks CSS transform interpolation and causes jarring pops. Keep all demo windows mounted in `.screen-overlay` and animate inactive windows to minimized dock coordinates or offscreen parking vectors with `opacity: 0` and `pointer-events: none`.
- **Avoid mixing layout reflow with transform animations:** Animating `left` and `top` directly on complex DOM trees triggers expensive browser layout recalculations every frame. Position windows at `left: 0; top: 0;` and animate `transform: translate3d(x, y, 0) scale(sx, sy)` so motion stays on the GPU compositor thread.
- **Lock coordinate ratios across viewport resizes:** If `.screen-overlay` changes pixel dimensions on window resize, hardcoded pixel offsets for cursor choreography and dock icons will drift off target. Express window rects and cursor waypoints as normalized percentages (`0..100%`) of `.screen-overlay` or scale a fixed-resolution virtual stage using a single CSS `--stage-scale` factor.
- **Prevent auto-play tour conflicts with manual user clicks:** When an `IntersectionObserver` or timer is advancing tour steps automatically, clicking a pill without clearing the active timer causes the stage to snap back mid-animation. Always clear active tour timers on any manual click or keypress event.
- **Missing user screenshots:** When the user only provides a product description without image assets, synthesize high-DPI SVG or HTML/CSS snapshot tiles inside each `.demo-window` frame rather than leaving placeholder boxes.

## Supporting Files and Agent Workflow

Read these files only as needed to keep the skill concise:

- [`references/REFERENCE.md`](references/REFERENCE.md) — scroll-progress controller, spring solver, normalized window/dock presets, and cursor choreography. The snippets are illustrative building blocks; adapt coordinates, state transitions, and lifecycle cleanup to the target app.
- [`assets/starter-template.html`](assets/starter-template.html) — runnable, dependency-free demonstration of workspace switching, layout presets, chapter selection, and simulated cursor drag.

1. Inspect the real product and define three meaningful feature demonstrations; do not copy example product names or visuals into unrelated deliverables.
2. Model each feature as a deterministic state with normalized window geometry. Use real screenshots when supplied, otherwise create clearly representative HTML/SVG demo surfaces.
3. Start with the HTML template or port its compositing/state patterns into the user's framework. Keep the source of truth for stage state in one controller.
4. Use the reference code for advanced motion, then add cleanup and interruption handling (replay, manual click, resize, visibility, and unmount).
5. Verify responsive layout, click/keyboard parity, `prefers-reduced-motion`, logical tab order, visible focus, and the absence of embedded demo video.

**Implementation boundary:** The starter template is a working educational blueprint, not a production app, and reference snippets must be integrated and tested rather than pasted together without wiring.
