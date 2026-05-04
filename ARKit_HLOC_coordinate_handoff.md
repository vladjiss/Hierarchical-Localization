# ARKit -> HLOC: camera data and coordinate conversion

## 1. What HLOC needs for a photo

For every query photo HLOC needs a camera model line:

```text
image_name PINHOLE width height fx fy cx cy
```

Example:

```text
frames/000123.jpg PINHOLE 1920 1440 1450.2 1451.0 960.4 720.1
```

This goes into `queries_with_intrinsics.txt`.

## 2. Where these values come from in ARKit

ARKit gives the camera intrinsics matrix:

```text
K = [ fx  0  cx
      0  fy  cy
      0   0   1 ]
```

Use:

```swift
let K = frame.camera.intrinsics
let res = frame.camera.imageResolution

let fx = K.columns.0.x
let fy = K.columns.1.y
let cx = K.columns.2.x
let cy = K.columns.2.y
let width = Int(res.width)
let height = Int(res.height)
```

Important: `fx/fy/cx/cy` must match the saved image pixels. If the image is resized, cropped, or rotated, the intrinsics must be transformed the same way.

## 3. What intrinsics solve

Intrinsics convert:

```text
pixel on photo -> 3D ray in camera coordinates
```

Formula for `PINHOLE`:

```text
x = (u - cx) / fx
y = (v - cy) / fy
ray = [x, y, 1]
```

So HLOC can take matched 2D pixels and known 3D map points, then solve the camera pose.

Intrinsics do not align ARKit world and HLOC/COLMAP world by themselves. They only describe how the camera sees.

## 4. What HLOC outputs

HLOC returns camera pose in the COLMAP/SfM map coordinate system:

```text
T_colmapCamera_from_colmapWorld
```

In simple terms:

```text
HLOC output = where this photo camera is inside the COLMAP map
```

This pose is not automatically in ARKit world coordinates.

## 5. How to align ARKit world and HLOC world

For several frames, we have both:

```text
ARKit camera center in ARKit world
HLOC camera center in COLMAP world
```

From HLOC pose:

```text
R_hloc, t_hloc = world -> camera
c_colmap = -R_hloc.T * t_hloc
```

From ARKit:

```text
T_arkitWorld_from_arkitCamera = frame.camera.transform
c_arkit = translation part of that matrix
```

Estimate one global similarity transform:

```text
c_colmap ~= scale * R_align * c_arkit + t_align
```

This is a standard point-set alignment problem, usually solved with Umeyama / Procrustes.

After this, any ARKit world point can be converted to COLMAP world:

```text
p_colmap = scale * R_align * p_arkit + t_align
```

And back:

```text
p_arkit = (1 / scale) * R_align.T * (p_colmap - t_align)
```

## 6. Camera-axis convention

ARKit camera axes and COLMAP camera axes are different:

```text
ARKit:  x right, y up,   z backward
COLMAP: x right, y down, z forward
```

Camera-axis conversion:

```text
S = diag(1, -1, -1)
```

Use this when comparing or converting full camera rotations. For camera centers only, this axis flip is not needed.

## 7. Bottom line

No blocker here:

```text
ARKit gives:
photo + intrinsics K + ARKit pose

HLOC accepts:
photo + PINHOLE width height fx fy cx cy

HLOC returns:
camera pose in COLMAP world

We add:
one global ARKit-world <-> COLMAP-world alignment transform
```

The intrinsics let HLOC understand the geometry of each photo. The alignment transform converts positions/poses between the ARKit world and the HLOC/COLMAP map world.
