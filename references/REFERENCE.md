# Reference Guide: Compositor, Spring Motion & Cursor Choreography

This reference provides the production JavaScript and CSS modules required to build a `backdrop-app.com`-grade interactive product demo stage without video playback.

## 1. Scroll-Pinned Dive Normalizer (`scroll-dive.js`)

Calculates `--ptn` (`0.0` to `1.0`) across a sticky viewport track and switches the stage to interactive mode once zoomed in:

```javascript
export function initScrollDive(diveContainer, stageEl) {
  let rafPending = false;
  const update = () => {
    rafPending = false;
    const rect = diveContainer.getBoundingClientRect();
    const scrollable = Math.max(1, diveContainer.offsetHeight - window.innerHeight);
    const progress = Math.max(0, Math.min(1, -rect.top / scrollable));
    stageEl.style.setProperty('--ptn', progress.toFixed(4));
    stageEl.dataset.interactive = progress >= 0.85 ? 'true' : 'false';
  };
  window.addEventListener('scroll', () => {
    if (!rafPending) {
      rafPending = true;
      requestAnimationFrame(update);
    }
  }, { passive: true });
  window.addEventListener('resize', update);
  update();
}
```

## 2. Damped Spring Physics Solver (`motion.js`)

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
    let last = performance.now();
    const tick = (now) => {
      const dt = Math.min((now - last) / 1000, 0.032);
      last = now;
      let movingCount = 0;

      for (const state of this.items.values()) {
        let active = false;
        for (const k of Object.keys(state.target)) {
          const cur = state.current[k];
          const tgt = state.target[k];
          const vel = state.velocity[k] || 0;
          const force = -this.stiffness * (cur - tgt) - this.damping * vel;
          const nextVel = vel + (force / this.mass) * dt;
          const nextCur = cur + nextVel * dt;

          if (Math.abs(nextVel) > 0.04 || Math.abs(tgt - nextCur) > 0.06) {
            active = true;
            state.velocity[k] = nextVel;
            state.current[k] = nextCur;
          } else {
            state.velocity[k] = 0;
            state.current[k] = tgt;
          }
        }
        state.onUpdate(state.current);
        if (active) movingCount++;
      }

      this.rafId = movingCount > 0 ? requestAnimationFrame(tick) : null;
    };
    this.rafId = requestAnimationFrame(tick);
  }
}
```

## 3. Snapshot Window & Dock Compositor (`desk.js`)

Maps `(project, layout)` state pairs to normalized `[x, y, w, h, scale, opacity, z]` targets for every window and animates inactive windows into their dock coordinates:

```javascript
export const DOCK_SLOTS = {
  xcode:    [36, 91, 6, 6],
  terminal: [43, 91, 6, 6],
  safari:   [50, 91, 6, 6],
  textedit: [57, 91, 6, 6],
  preview:  [64, 91, 6, 6]
};

export const LAYOUT_RECTS = {
  'main-stack': [
    [2.5, 8, 56, 77],
    [60,  8, 37.5, 37],
    [60, 48, 37.5, 37]
  ],
  'side-by-side': [
    [2.5, 8, 46.5, 77],
    [51,  8, 46.5, 77],
    [16, 13, 68,   68]
  ],
  'grid': [
    [2.5, 8, 46.5, 37],
    [51,  8, 46.5, 37],
    [2.5, 48, 95,  37]
  ],
  'focus': [
    [11,  7, 78,   79],
    [15, 11, 70,   71],
    [19, 15, 62,   63]
  ],
  'keep-layout': [
    [5,  10, 52,   68],
    [36, 16, 48,   64],
    [20, 22, 54,   62]
  ]
};
```

## 4. Autonomous Cursor Drag-to-Notch Sequencer (`notchboard.js`)

Executes a multi-step timeline that moves `.demo-cursor`, grabs a window header, drags it toward `.os-notch`, expands the Notchboard project shelf, drops the window into a target project, and re-tiles the desktop:

```javascript
export class NotchDragSequencer {
  constructor({ stageEl, cursorEl, notchEl, onReassignWindow }) {
    this.stageEl = stageEl;
    this.cursorEl = cursorEl;
    this.notchEl = notchEl;
    this.onReassignWindow = onReassignWindow;
    this.timers = [];
  }

  stop() {
    this.timers.forEach(clearTimeout);
    this.timers = [];
    this.cursorEl.dataset.state = 'hidden';
    this.notchEl.dataset.open = 'false';
    this.notchEl.dataset.hoverSlot = '';
    this.stageEl.dataset.draggingWin = '';
  }

  play() {
    this.stop();
    const { width, height } = this.stageEl.getBoundingClientRect();
    const at = (ms, fn) => this.timers.push(setTimeout(fn, ms));

    at(50, () => {
      this.cursorEl.dataset.state = 'visible';
      this.cursorEl.style.transform = `translate3d(${width * 0.25}px, ${height * 0.18}px, 0)`;
    });

    at(750, () => {
      this.cursorEl.dataset.state = 'grabbing';
      this.stageEl.dataset.draggingWin = 'safari';
    });

    at(1250, () => {
      this.cursorEl.style.transform = `translate3d(${width * 0.5}px, ${height * 0.04}px, 0)`;
      this.notchEl.dataset.open = 'true';
      this.notchEl.dataset.hoverSlot = 'personal';
    });

    at(2300, () => {
      this.cursorEl.dataset.state = 'visible';
      this.stageEl.dataset.draggingWin = '';
      this.notchEl.dataset.open = 'false';
      this.onReassignWindow?.('safari', 'personal');
    });
  }
}
```

## Integration and Validation Notes

These modules are architectural samples, not a complete copy-and-paste application. Use the stage's actual dimensions for all cursor waypoints and store state in one controller. Couple the cursor's drop action to the same state update used by manual controls.

- Use a stable logical coordinate system or percentages for windows and dock targets; derive pixel positions after measuring the stage.
- The animator's first `setTarget` initializes at the target without an entrance animation. Initialize from the desired start position before sending an animated target.
- `NotchDragSequencer.stop()` clears pending steps; also invoke it before manual state changes, on resize, and on unmount. Remove global event listeners and cancel any active animation frames when the owning component unmounts.
- Disable automatic animation when `prefers-reduced-motion: reduce` matches; keep all product functions available.
- Validate every demo state with direct clicks and keyboard access, including rapid replay, switching layouts mid-gesture, and small-screen resizing.
