Build SPERR on Windows in VS Code
Recommended build

Use the Visual Studio 2022 build. It was the clean path that worked.

Prerequisites

Install:

Visual Studio 2022 with Desktop development with C++
CMake
Git

Verify in PowerShell:
  cmake --version
  git --version

1. Open the SPERR folder in VS Code
  File → Open Folder → C:\dev\SPERR
Open a terminal in VS Code.

2. Create a build folder
  cd C:\dev\SPERR
  mkdir build_vs2022
  cd build_vs2022

3. Configure with CMake
  cmake -G "Visual Studio 17 2022" -A x64 ..
If you want the test executables too:
  cmake -G "Visual Studio 17 2022" -A x64 -DBUILD_TESTING=ON ..

4. Build
  cmake --build . --config Release

This should produce the main executable at:
C:\dev\SPERR\build_vs2022\bin\Release\sperr3d.exe

You can check:
  dir C:\dev\SPERR\build_vs2022\bin\Release\*.exe

5. Test that SPERR runs
  cd C:\dev\SPERR\build_vs2022\bin\Release
  .\sperr3d.exe
  
If it runs, it should at least prompt for input or show a usage-related message.


Optional: build with tests
If you want the extra test tools:

  cd C:\dev\SPERR
  mkdir build_vs2022_tests
  cd build_vs2022_tests
  cmake -G "Visual Studio 17 2022" -A x64 -DBUILD_TESTING=ON ..
  cmake --build . --config Release


That may generate tools like:
bitstream.exe
dwt.exe
outlier_coder.exe
speck2d_flt.exe
speck3d_flt.exe
sperr_helper.exe
Important note about MinGW

You can build SPERR with MinGW, but in this project it caused:
DLL issues
pseudo-relocation errors
runtime mismatch problems
So for this setup, prefer:
Visual Studio 2022 build = recommended
MinGW build = not recommended


Full build workflow summary
  cd C:\dev\SPERR
  mkdir build_vs2022
  cd build_vs2022
  cmake -G "Visual Studio 17 2022" -A x64 ..
  cmake --build . --config Release
  cd .\bin\Release
  .\sperr3d.exe
Troubleshooting
cmake not recognized
Make sure CMake is installed and on PATH.
Build succeeds but EXE missing
Check:
  dir C:\dev\SPERR\build_vs2022\bin\Release\*.exe

MinGW build crashes
Do not chase that path further unless you really need GCC specifically. Use the VS2022 build.

Build looks stuck near the end
Do not press Ctrl+C. Let linking finish.

Output you should expect
Main executable:

C:\dev\SPERR\build_vs2022\bin\Release\sperr3d.exe

That is the one to use for:
compressing .raw
decompressing .stream




📘 README — SPERR Video Compression Pipeline (VS Code)
📌 Overview

This project demonstrates how to:

Convert a video → RAW (float32)
Compress using SPERR
Decompress back to RAW
Convert RAW → MP4 for visualization
⚙️ Prerequisites

Install the following:

1. Visual Studio Code
Install Python extension
2. Python (≥ 3.10 recommended)
python --version
3. FFmpeg

Verify:

ffmpeg -version
4. SPERR (already built)

You should have:

C:\dev\SPERR\build_vs2022\bin\Release\sperr3d.exe
📁 Project Structure
SPERR/
├── python_files/
│   ├── video_to_raw.py
│   ├── raw_to_video.py
│   ├── video_small.raw
│   ├── video_small_100.raw
│   ├── video_small_100.decomp.raw
│   └── output.mp4
├── build_vs2022/
│   └── bin/Release/sperr3d.exe
🚀 Step 1 — Open in VS Code
File → Open Folder → C:\dev\SPERR
🐍 Step 2 — Setup Python Environment

Open terminal in VS Code:

cd python_files
python -m venv .venv
.\.venv\Scripts\activate
pip install numpy opencv-python

Select interpreter:

Ctrl + Shift + P → Python: Select Interpreter → .venv
🎥 Step 3 — Convert Video → RAW

Example script (video_to_raw.py):

import cv2
import numpy as np

cap = cv2.VideoCapture("input.mp4")

frames = []
max_frames = 100  # limit to avoid memory issues

count = 0
while cap.isOpened() and count < max_frames:
    ret, frame = cap.read()
    if not ret:
        break
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    frames.append(gray.astype(np.float32))
    count += 1

cap.release()

volume = np.stack(frames)  # (frames, H, W)
volume.tofile("video_small_100.raw")

print("Saved RAW:", volume.shape)

Run:

python video_to_raw.py
📦 Step 4 — Compress with SPERR
cd C:\dev\SPERR\build_vs2022\bin\Release

$in     = "C:\dev\SPERR\python_files\video_small_100.raw"
$stream = "C:\dev\SPERR\python_files\video_small_100.stream"

.\sperr3d.exe -c --ftype 32 --dims 100 270 480 --pwe 1e-4 --bitstream $stream $in
📤 Step 5 — Decompress
$out = "C:\dev\SPERR\python_files\video_small_100.decomp.raw"

.\sperr3d.exe -d --ftype 32 --dims 100 270 480 --bitstream $stream $out
🎬 Step 6 — Convert RAW → MP4 (Correct Way)
🔥 Important: Normalize float32 → uint8

Run:

python -c "import numpy as np; a=np.fromfile(r'C:\dev\SPERR\python_files\video_small_100.decomp.raw',dtype=np.float32).reshape(100,270,480); amin=a.min(); amax=a.max(); b=((a-amin)/(amax-amin)*255).astype(np.uint8); b.tofile(r'C:\dev\SPERR\python_files\video_small_100_uint8.raw')"
Convert to video:
ffmpeg -f rawvideo -pixel_format gray -video_size 480x270 -framerate 30 `
-i "C:\dev\SPERR\python_files\video_small_100_uint8.raw" `
"C:\dev\SPERR\python_files\output.mp4"
🧪 Optional — Verify Reconstruction Error
python -c "import numpy as np; a=np.fromfile(r'C:\dev\SPERR\python_files\video_small_100.raw',dtype=np.float32); b=np.fromfile(r'C:\dev\SPERR\python_files\video_small_100.decomp.raw',dtype=np.float32); print('Max error:', np.max(np.abs(a-b)))"
⚠️ Common Issues
1. ❌ Static / Noise Video

Cause:

Wrong pixel format (gray instead of grayf32le)

Fix:

Always normalize before converting to video
2. ❌ “File does not exist”

Cause:

Using Join-Path incorrectly

Fix:

$in = "C:\full\path\file.raw"
3. ❌ Memory Error (Huge video)

Cause:

Too many frames loaded into RAM

Fix:

max_frames = 100
4. ❌ SPERR crashes

Cause:

Wrong dimensions or missing DLLs

Fix:

Ensure dims match RAW exactly
Use VS2022 build (you already fixed this)
🧠 Key Concepts
Concept	Meaning
RAW	No header, pure binary
float32	4 bytes per pixel
dims	(frames, height, width)
SPERR	Scientific compression (not image codec)
🚀 Next Steps (if you want to push further)
Chunk large videos (avoid RAM issues)
Compare SPERR vs H.264 (PSNR, bitrate)
Visualize error heatmaps
Automate full pipeline

If you want, I can convert this into:

📄 a polished PDF report
⚙️ a one-click Python pipeline script
📊 experiment framework (for your class)

Just say the word.