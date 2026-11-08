# Ghost & Invisibility Effects on Live Video

Real-time video effects built with **OpenCV** and **background subtraction**, running on a live webcam.

## Effects
- **Ghost effect** – a faded, semi-transparent copy of you trails behind you, while you stay solid.
- **Invisibility effect** – your whole body disappears and the scene looks empty.

## How it works
1. Capture an empty background (median of 60 frames)
2. Compare each frame with the background (`cv2.absdiff`) to find the person
3. Clean the mask (morphology, filled contours, dilation, smoothing over frames)
4. **Ghost:** blend the frame from 30 frames ago where you used to be
5. **Invisibility:** replace your pixels with the saved background

I also compared my approach with OpenCV's `MOG2` background subtractor (press `m` to switch).

## Run it
pip install opencv-python numpy
python ghost_effect.py

Step out of the frame for the first 2–3 seconds so it can learn the background. Press `ESC` to quit.

## What I learned
- A fixed background works best with a still camera, while MOG2 adapts to changes but slowly absorbs people who stand still
- Handheld camera movement breaks both methods, since background subtraction assumes a fixed camera