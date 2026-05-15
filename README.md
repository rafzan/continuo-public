# Continuo

**Pen plotter scribble art generator**

<img width="939" height="847" alt="Captura de Tela 2026-05-15 às 01 03 29" src="https://github.com/user-attachments/assets/922aaddf-3e58-4c43-97db-7001edc72ce0" />
<img width="940" height="848" alt="Captura de Tela 2026-05-15 às 01 02 23" src="https://github.com/user-attachments/assets/602cc852-8ad2-4d16-af21-c66bf5f62846" />

Continuo converts photographs and images into SVG line art optimised for pen plotters. It separates an image into tonal layers, runs one of several rendering engines over each layer, and exports a plotter-ready SVG — either as a single-colour drawing or a full CMYK multi-pen composition.

---

### Canvas settings

Found at the top of the left panel. Set your paper size (A4, A3, Letter, or custom), margin, pen width (in mm), and working resolution. These drive the SVG viewport and physical scaling so the output is ready to plot at the correct size.

### Colour mode

**Single** — one ink colour, the image is converted to greyscale.

**CMYK** — the image is separated into four channels (Cyan, Magenta, Yellow, Black) using Under Colour Removal (UCR). Each channel is rendered independently and exported as a separate SVG layer. Each channel has its own colour swatch, which can be recoloured live after rendering without re-running the engine.

**Absorb black into CMY** (CMYK only) — merges the K channel into C, M and Y so that black areas appear as a dense overlay of all three inks. Produces richer darks with only three pen passes instead of four.

TBD: Pen sets. Grays, Blues, etc.

### Step pills

After a run, a row of pills appears above the canvas: Input → intermediate steps → final result. Click any pill to jump to that stage. For CMYK, clicking a channel pill reveals a sub-row with that channel's individual steps.

### Layer panel

Appears after a CMYK render. Each row represents one ink channel and offers:
- **Eye** — toggle visibility
- **Opacity** slider — adjust blending weight live
- **Colour swatch** — recolour the channel instantly without re-rendering
- **↺ Regen** — re-run just this channel with current settings

### Presets

Save and load named parameter sets (stored as `.json` files in `~/.config/continuo/presets/`). Useful for keeping consistent settings across different images.

### Session save / load

**💾 Save session** and **📂 Load session** store the complete state of the app — all parameters, rendered channel images, and layer panel settings — into a single `.continuo` file. Sessions are self-contained: the rendered images are embedded as Base64 so they do not depend on the original image file being present.

---

## Engines

### Circular — Chiu 2015

Based on the algorithm from *Chiu et al., 2015*. Samples the image using SLIC superpixels, builds a nearest-neighbour path through the samples, then draws tight spiral circles at each sample point, sized by local luminance. Produces the classic dense-scribble aesthetic.

| Parameter | Description |
|-----------|-------------|
| Density scale | Global dot density multiplier |
| Circle scale | Radius multiplier for each circle |
| Tilt | Angle of the per-circle spirals |
| Regions | Number of SLIC superpixel regions |
| Hard edges | Constrain circles to stay within region boundaries |
| Focus sampling | Only place samples in areas darker than the cutoff threshold |
| Stroke mode | **Spiral** (default) or **Bézier** — Bézier replaces circles with smooth Catmull-Rom curves through the sample path, producing a flowing gestural line instead of spirals |

**Bézier sub-params:** Smoothing (control-point stride) and Tension (curve tightness, 0.1–1.0).

---

### Haystack — Oskay 2017

A TSP-style tour algorithm. Builds a single continuous path that covers the dark areas of the image by iteratively extending the tour toward the highest-value unvisited area. The output is one unbroken line — zero pen lifts.

| Parameter | Description |
|-----------|-------------|
| Segment length | Physical length of each tour step (mm) |
| Blur factor | Smoothing applied to the canvas before touring |
| Opacity factor | How aggressively the tour depletes visited areas |
| Max segments | Hard cap on tour length |

---

### Hatching — Directional

Directional stroke hatching. Traces strokes perpendicular to the local image gradient, so lines naturally follow the contours of the subject. Produces a woodcut or engraving aesthetic.

| Parameter | Description |
|-----------|-------------|
| Line spacing | Pixels between scan lines |
| Min / Max length | Stroke length limits |
| Threshold | Ink cutoff — skip pixels brighter than this |
| Smoothing | Blur kernel for gradient computation |
| Angle (°) | Fixed angle override; −1 = follow local gradient |
| Crosshatch | Add a second pass at +90° |

---

### Squiggly — Scan lines

Horizontal scan rows rendered as sine waves. Amplitude is modulated by local luminance: dark areas produce tall, expressive waves; bright areas produce a flat line or nothing at all.

| Parameter | Description |
|-----------|-------------|
| Row spacing | Distance between scan rows |
| Frequency | Sine cycles across the image width |
| Max / Min amplitude | Wave height range |
| Threshold | Rows brighter than this are skipped |
| Angle (°) | Rotate the scan direction |
| Phase drift | Random phase offset per row for an organic feel |

---

### Circles — Halftone

A grid of circles sized by local luminance, producing a halftone dot screen. Optional concentric rings fill darker cells with multiple nested circles.

| Parameter | Description |
|-----------|-------------|
| Cell size | Grid cell size in pixels |
| Min / Max radius | Circle radius range |
| Threshold | Skip cells brighter than this |
| Jitter | Random offset applied to each circle centre |
| Concentric rings | Draw multiple nested rings per cell |
| Ring spacing | Distance between concentric rings |

---

### Contours — Fast March

Uses the **Fast Marching Method** (Sethian 1996) to propagate a wave from one or more seed points. The wave travels slowly through dark areas and quickly through bright areas, so iso-contours of the arrival-time field pack densely in shadows and spread apart in highlights — producing topographic contour lines that organically follow the subject's form.

**Seed modes:**
- **Single** — click the preview to place one seed; each click moves it
- **Multi** — click to accumulate multiple seeds; double-click to clear

| Parameter | Description |
|-----------|-------------|
| Seed shape | Circle, Triangle, Square, Diamond, Pentagon, Hexagon, Heptagon, Octagon, Star, Cross |
| Seed radius | Size of the seed region in pixels |
| Angle (°) | Rotate the seed shape |
| Skew X / Y | Stretch the seed shape horizontally or vertically |
| Star points | Number of star points (Star shape only) |
| Inner ratio | Ratio of inner to outer radius (Star shape only) |
| Contour spacing | T-field interval between extracted contour lines |
| Min speed | Floor for the propagation speed map |
| Smooth | Pre-blur kernel for the speed map |
| Threshold | Brightness cutoff for ignoring bright areas |
| Ignore white | Contours stop at areas brighter than the threshold |
| Hard seed | Prepend the exact geometric outline of the seed shape as the first stroke |

---

### Boustrophedon Fill

Fills connected ink regions with a continuous back-and-forth serpentine stroke — the pen never lifts inside a region. Lines are generated directly at the specified angle using parametric geometry (no image rotation). Each connected component in the thresholded image becomes one continuous stroke.

**Single-channel, multi-layer mode:** up to 4 hatching passes are stacked at different thresholds and angles, derived automatically from the base threshold. The angles are maximally separated within the half-circle.

| Layers | Thresholds | Angles |
|--------|-----------|--------|
| 1 | T | 0° |
| 2 | T, T/2 | 0°, 90° |
| 3 | T, 2T/3, T/3 | 0°, 60°, 120° |
| 4 | T, 3T/4, T/2, T/4 | 0°, 45°, 90°, 135° |

**CMYK mode:** each channel is rendered at its traditional halftone screen angle (C=15°, M=75°, Y=0°, K=45°).

| Parameter | Description |
|-----------|-------------|
| Threshold | Ink cutoff — pixels darker than this are filled |
| Line spacing | Distance between scan lines |
| Base angle (°) | Starting angle for the scan lines |
| Smooth | Pre-blur kernel |
| Layers | Number of stacked passes (1–4, single-channel only) |
| Connected | Enable boustrophedon (serpentine) connection |
| Min connect angle | Minimum hairpin angle relative to scan direction (default 35°) |
| Max connect angle | Maximum hairpin angle relative to scan direction (default 145°) |

The connection angles are relative to the scan line direction — 90° is a perfect perpendicular hairpin. Values outside the range cause the stroke to break, preventing ugly diagonal leaps between disconnected regions.

**Step pills:** Input → Threshold (B&W posterized mask) → L1 → L2 → L3 → L4

---

## SVG export

Click **⬇ Save SVG…** after a render. Single-colour exports are a flat SVG. CMYK exports produce a layered SVG with one `<g>` per channel, each tagged with `mix-blend-mode: multiply` so the layers composite correctly in Inkscape, Illustrator, or a web browser.

---

## Canvas viewer

**Mouse wheel** — zoom 50%–200%, centred on the cursor position.
**Click + drag** — pan the canvas.
**1:1 button** — reset zoom and pan to default.

---

## Keyboard / workflow tips

- Run by clicking **▶ Run** or pressing **Enter** (when focus is not in a text field).
- Click any step pill to inspect that stage before the run finishes — useful for aborting early if the threshold or sampling looks wrong.
- In CMYK mode, use the **Regen** button on a single layer row to re-render just that channel after adjusting its parameters, without re-running all four.
- The **Threshold** step pill in Boustrophedon Fill shows exactly which pixels will be filled before committing to the full run.

---

*Continuo is a personal project. Engines are based on published algorithms by Chiu et al. (2015), Oskay (2017), Wygonik / Evil Mad Scientist Laboratories (SquiggleDraw), and Sethian (1996 Fast Marching Method).*
