# hearth_companions

A small, shared library of **3D voice-companion avatars** for the Hearth
projects — the little "Clippy in the corner" characters that lip-sync to an
assistant's voice, blink, and emote while you talk to them.

Each companion is a `.glb` 3D model plus a tiny `.avatar.json` sidecar that
tells a renderer how to bring it to life. They're stored here, versioned, so
any project can pull them in and let the user pick one.

Three companions ship today:

| ID          | Name     | What it is                                             |
| ----------- | -------- | ------------------------------------------------------ |
| `floppy`    | Floppy   | A floppy disk — the retro save-icon mascot.            |
| `orb`       | Orb      | A floating orb — minimal, abstract, calm.              |
| `iron-mask` | Iron Man | An Iron-Man-style helmet with a metallic PBR finish.   |

## Layout

```
hearth_companions/
├── manifest.json            # index of every companion + its versions
├── floppy/
│   └── v1/
│       ├── floppy.glb
│       └── floppy.avatar.json
├── orb/
│   └── v1/
│       ├── orb.glb
│       └── orb.avatar.json
└── iron-mask/
    └── v1/
        ├── iron-mask.glb
        └── iron-mask.avatar.json
```

**Convention**

- One folder per companion, named by its `id` (lowercase, kebab-case).
- Inside it, one folder per **version**: `v1`, `v2`, … The highest is the
  latest. A companion with no version subfolder is treated as `v1`.
- Inside a version folder, the model and its sidecar are both named after the
  companion id: `<id>.glb` and `<id>.avatar.json`. A renderer that knows the
  id can find both.
- `manifest.json` is the machine-readable index — consumers read it to
  enumerate what's available (and which versions) without walking the tree.

## The avatar contract (`<id>.avatar.json`)

```jsonc
{
  "name": "Floppy",          // display name
  "mode": "face",            // "face" = has morph targets; "prop" = rigid, animated by transform
  "morphs": {                // maps contract slots -> the GLB's actual morph-target names
    "mouthOpen": "mouthOpen",// driven by the RMS amplitude of the assistant's voice (lip-sync)
    "blink": "blink",        // driven by a randomized 2–7s blink timer
    "mood_happy": "mood_happy",
    "mood_think": "mood_think",
    "mood_alert": "mood_alert"
  },
  "scale": 1,                // uniform scale applied to the scene root
  "cameraDistance": 5        // camera distance from origin (default 3)
}
```

- **`mode: "face"`** — the renderer walks the GLB's mesh graph and drives the
  named morph targets: `mouthOpen` follows the voice, `blink` fires on a
  timer, and the `mood_*` morphs ease in/out with conversation state
  (listening / thinking / speaking / happy / alert). `mouthOpen` and `blink`
  are required; the moods are optional.
- **`mode: "prop"`** — no rig assumed. The renderer animates the whole object
  instead: a speech-amplitude wobble + scale pulse, a happy spin, an alert
  shake, a listening lean.

If a GLB or manifest is missing or fails to load, a well-behaved renderer
should fall back to a placeholder (e.g. a shaded sphere that still pulses to
the voice) rather than showing a blank canvas.

## Adding a companion

1. Make a folder named after the new id: `my-buddy/`.
2. Add `my-buddy/v1/my-buddy.glb` and `my-buddy/v1/my-buddy.avatar.json`.
3. Add an entry to `manifest.json`.

## Adding a new version of an existing companion

1. Add `iron-mask/v2/iron-mask.glb` and `iron-mask/v2/iron-mask.avatar.json`.
2. Bump `latest` and append `"v2"` to that companion's `versions` in
   `manifest.json`.

Old versions stay put, so consumers can pin or offer a version picker.

## Who consumes this

Built for the Hearth voice-companion renderer (three.js / react-three-fiber):
it loads `<id>.glb` with drei's `useGLTF`, fetches `<id>.avatar.json`
alongside, resolves the `morphs` map to concrete mesh + morph-influence
indices, and drives them every animation frame — `mouthOpen` from an
`AnalyserNode` on the assistant's audio track, `blink` from a timer, and the
`mood_*` morphs from the conversation state machine.

The **talk-to-diagram** test project is the first consumer: you talk, a
companion in the corner listens and speaks back, and it draws diagrams on a
canvas from what you say.
