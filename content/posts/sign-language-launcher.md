+++
title = "Sign Language Launcher: Opening Apps with Hand Gestures Using MediaPipe and Python"
date = 2026-11-18
draft = false
description = "A computer vision project that reads ASL letters from a webcam and launches an app for each one, plus the five problems I had to debug along the way."
tags = ["computer-vision", "python", "mediapipe", "machine-learning"]
+++

I built a program that watches my webcam, recognizes a letter in American Sign Language, and opens an application for it. Sign **S** and Spotify opens. Sign **C** and the calculator opens. Sign **Y** and YouTube opens.

[Insert Video Demo Here]

The code is on GitHub: [CV-ASL-Detection](https://github.com/ketatamohamed4/CV-ASL-Detection). This post explains how it works, but most of it is about what broke and how I fixed it, because that's where I learned the most.

## How it works

The pipeline has four steps:

1. **Webcam.** OpenCV gives me a picture many times per second.
2. **Hand landmarks.** For each picture, MediaPipe finds my hand and returns 21 points: the wrist, the knuckles, and the fingertips.
3. **Normalization.** I clean up the points so the hand's position in the frame and its distance from the camera don't matter. The wrist becomes the center, and everything is divided by the size of the hand. Only the *shape* of the hand is left.
4. **Classification.** A trained model looks at those numbers and guesses the letter.

I don't give the model the picture itself. The 21 points describe the shape of the hand without the background, the lighting, or the skin color, so the model has far less to get confused by.

## Choosing the data

My professor advised me not to train on photos I took myself. A good dataset needs variety (different people, hands, lighting, and backgrounds), and I can't produce that alone. So I trained on the public [ASL Alphabet dataset from Kaggle](https://www.kaggle.com/), using MediaPipe to extract landmarks from its images and a Random Forest to classify them.

## Troubleshooting: what went wrong

### Problem 1: Confidence stuck at 0.2

**Symptom.** While recording live, the model's confidence never rose above about 0.2, but my threshold for accepting a letter was 0.8. So nothing was ever detected, and I couldn't test the rest of the program.

**What I did.** I lowered the threshold temporarily so I could keep testing the other parts of the code. This was a workaround, not a fix. The low confidence was a sign that the live data looked different from the training data. That led directly to Problem 2.

**Where it ended up.** After fixing the underlying issues and training on more letters, the threshold works at 0.45.

### Problem 2: The model only worked in one orientation

**Symptom.** The model recognized a letter only when the hand was oriented the way it appeared in the training images.

**Cause.** The model learns what it is shown. I can't know whether a user's camera is mirrored or not, and some letters look different mirrored.

**Fix.** I mirrored every image in the dataset and added the mirrored copies, doubling the training data. The model now recognizes letters in either orientation. The live video is always mirrored before it goes to MediaPipe, so there's no setting for the user to get wrong.

### Problem 3: The app launched over and over

**Symptom.** After recognizing a letter and opening the app, the program went straight back to detecting. My hand was still in the frame, so it launched the app again, and again, until the program crashed.

**Fix.** A cooldown. After an app opens, the program waits 5 seconds before another one can open.

```python
COOLDOWN_SECONDS = 5

if now - last_launch < COOLDOWN_SECONDS:
    status = "cooldown..."
elif now - hold_start >= HOLD_SECONDS:
    launch(stable)
    last_launch = now   # remember when the app was launched
```

### Problem 4: Accidental triggers

**Symptom.** A letter could be recognized for a single moment, for example while my hand passed through a shape on the way to another one, and the program would act on it.

**Fix.** Two rules. A letter is only accepted if it appears in most of the last 10 frames, and I have to hold it for 1.5 seconds before anything opens. I also added a `DRY_RUN` mode that only prints what the program would open, so I could check which letters it reads before letting it launch anything.

```python
HOLD_SECONDS = 1.5

if stable != hold_letter:   # a different letter appeared
    hold_letter = stable
    hold_start = now        # restart the hold timer

# a launch only happens after the letter has been held long enough:
elif now - hold_start >= HOLD_SECONDS:
    launch(stable)
```

### Problem 5: Making it work on Windows and Linux

I wanted it to run on both without anyone editing the code. The program detects the operating system with `sys.platform` and picks the right approach.

- **Camera.** DirectShow on Windows, V4L2 on Linux.
- **File paths.** Handled so the same code runs on both.
- **Opening apps.** A command on Windows, `open -a` on macOS, and `gtk-launch` on Linux.
- **Linux app IDs.** `gtk-launch` needs the app's `.desktop` ID, which is why the Linux entries look like `com.spotify.Client` instead of `spotify`. To find the right ID for an app:

```bash
ls /usr/share/applications/ /var/lib/flatpak/exports/share/applications/ ~/.local/share/applications/ 2>/dev/null | grep -i spotify
```

- **Python version.** The project uses MediaPipe 0.10.14 because newer releases removed `mp.solutions`, which the project is built on. That version has no package for Python 3.12 or 3.13, so on a recent distro like Fedora I had to install Python 3.11 and build the virtual environment with it.

## Results and limitations

- **Accuracy.** My first model, trained on six letters (A, B, C, L, S, Y), reached 99.5% on the test data. That number is too optimistic. The test data comes from the same dataset as the training data, so it says little about how the model behaves on my own webcam, where it was much less sure of itself.
- **Unknown signs.** When I sign something the model doesn't know, the program shows "unknown" instead of opening anything.
- **False triggers.** In a `DRY_RUN` test, I saw no false triggers with the final settings. That is a small test, not a proper benchmark.
- **Platforms.** Tested on Windows 11 and Fedora with KDE Plasma. The macOS code is written but untested.

## What I'd improve next

- Measure false launches properly instead of eyeballing them.
- Test with other people's hands, not only mine.
- Try removing the z coordinate from the landmarks.

## Run it yourself

The setup steps, including the Python 3.11 note, are in the [README](https://github.com/ketatamohamed4/CV-ASL-Detection). Start with `DRY_RUN = True` so you can check the letters are read correctly before anything opens.

## What I learned

A model that scores 99.5% on its test set can still fail on a real webcam, so you have to test in the conditions where it will be used. Most of the work wasn't the model. It was the surrounding logic: cooldowns, hold times, and cross-platform details. And a temporary workaround, like lowering the threshold, is useful for debugging, but it should point you toward the real cause.
