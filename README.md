# Ghost & Invisibility Effects on Live Video

Real-time video effects on a live webcam stream using **classical background subtraction** in OpenCV, with an experimental comparison against OpenCV's **MOG2** (Gaussian Mixture Model) subtractor.

| Effect | Description |
|---|---|
| **Ghost** | A semi-transparent echo of the person trails a few moments behind them, while the real person stays solid. |
| **Invisibility** | The person is removed from the scene with no holes, outline or flicker. |

---

## Pipeline

    Webcam frame → flip → foreground segmentation → mask refinement → effect compositing → display

### 1. Background model
A static background `B` is estimated as the **per-pixel temporal median** of 60 frames captured while the scene is empty:

    B(x, y) = median{ I₁(x, y), …, I₆₀(x, y) }

The median is robust to noise and brief motion during capture.

### 2. Foreground segmentation
**Our approach** (frame differencing against `B`):

    D = | Gσ(I) − Gσ(B) |          (Gaussian blur, 5×5)
    M = 1  if gray(D) > 30,  else 0

**MOG2 (comparison):** `cv2.createBackgroundSubtractorMOG2(history=500, varThreshold=25, detectShadows=True)`
- Warm-started on the same 60 background frames before the person enters
- `learningRate=0.001`, so a stationary person is absorbed into the model slowly
- Shadow pixels (value 127) are removed by thresholding the mask at 200

### 3. Mask refinement
| Step | Operation | Purpose |
|---|---|---|
| Opening | `MORPH_OPEN`, 5×5, 2 iterations | Remove salt noise |
| Closing | `MORPH_CLOSE`, 5×5, 2 iterations | Bridge small gaps |
| Contour filling | `findContours` → keep area > 3000 px → `drawContours(..., -1)` | Remove small blobs, **fill holes** in the body |
| Dilation | 5×5, 4 iterations (invisibility) | Cover the body boundary, **no outline** |
| Temporal smoothing | `accumulateWeighted(M, S, α=0.5)` then threshold | Exponential moving average of the mask, **no flicker** |

### 4. Compositing
**Invisibility:** masked replacement with the background

    Out = B ⊙ M + I ⊙ (1 − M)

**Ghost:** a ring buffer (`collections.deque`, `maxlen=30`) stores past frames `I₍t−k₎` and masks `M₍t−k₎`. The ghost region is where the person *was* but *is not now*:

    G = M₍t−30₎ ∧ ¬M₍t₎

`G` is Gaussian-blurred (21×21) into a soft alpha matte `α`, scaled by opacity 0.5 and zeroed inside `M₍t₎` so the real person stays opaque:

    Out = I₍t₎ · (1 − α) + I₍t−30₎ · α

---

## Experiments

Challenge tested: **handheld camera** (camera motion).

| Condition | Our approach | MOG2 |
|---|---|---|
| Static camera, ghost | Clear trailing echo, solid subject | Similar quality |
| Static camera, invisibility | Solid hole-free mask, faint silhouette | More holes, so hands and clothing leak through |
| Handheld, ghost | Background regions get ghosted and smeared | Fewer background artifacts |
| Handheld, invisibility | Large parts of the scene are misclassified as foreground | Person only partly detected |

### Observations
- A **fixed median background** gives the cleanest mask with a static camera, but it can't adapt to motion or lighting changes.
- **MOG2** adapts over time, but with a low learning rate it struggles to resegment the scene after camera motion, and with a high rate it absorbs stationary people.
- **Camera auto-exposure** shifts the global intensity when the person enters, which leaves a faint silhouette and in some cases saturates the whole mask.
- Both methods assume a **static camera**. Handling camera motion would need frame registration (for example, homography alignment) before differencing.

---

## Usage

    pip install opencv-python numpy
    python ghost_effect.py
    python invisibility_effect.py

| Key | Action |
|---|---|
| `m` | Switch between our approach and MOG2 |
| `ESC` | Quit |

Stay out of the frame for the first 2 to 3 seconds while the background is captured.

## Tech stack
Python · OpenCV · NumPy
