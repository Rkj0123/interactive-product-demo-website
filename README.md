# Interactive Product Demo Website

An [Agent Skills](https://agentskills.io/specification) skill for creating interactive product marketing demos from persistent UI snapshot layers, DOM transitions, animated cursor choreography, and click/keyboard-controlled chapters—**without screen-recording video**.

Inspired by the interaction style of [Backdrop](https://backdrop-app.com/); this repository is an independent teaching/implementation resource and is not affiliated with Backdrop.

## Contents

```text
interactive-product-demo-website/
├── SKILL.md                   # Agent instructions and trigger conditions
├── references/
│   └── REFERENCE.md           # Spring motion, scroll dive, layout, cursor examples
└── assets/
    └── starter-template.html  # Open directly in a browser
```

## Using the skill

Add this repository's directory to your AI coding agent's configured skills directory or make its `SKILL.md` available to the agent. Ask for a scroll-driven interactive product demo, product landing page, or non-video UI showcase. The agent should read the core instructions first and pull in reference files as needed.

Example prompt:

> Build a product landing page for my macOS window manager. Show three simulated workspaces, several tiling layouts, and a replayable drag-to-menu-bar interaction. The product demo should use live DOM and screenshot layers, not an MP4. Follow the interactive-product-demo-website skill.

## Preview

Open [`assets/starter-template.html`](assets/starter-template.html) directly in a modern browser. Choose workspace pills (or press 1/2/3), change the layout, select the demo chapters, and replay the animated notch drag. The template uses no build tools or external dependencies.

The template is a **starting blueprint**; adapt its labels, assets, workflows, responsive layouts, state controller, and motion fidelity to the actual product. For multi-chapter implementations, follow the deeper [reference guide](references/REFERENCE.md) and test interruption/replay behavior.

## Skill goals

- Persistent layered product windows that rearrange without unmount flashes.
- Normalized stage coordinates and spring-compatible motion.
- Scroll-pinned hero dive and chapter-driven demos.
- Keyboard/click parity, reduced-motion support, and responsive fallbacks.
- No embedded demo video.

## License

MIT (as declared in `SKILL.md`).
