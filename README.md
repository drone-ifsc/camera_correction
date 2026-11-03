# camera_correction

Camera calibration data (intrinsics + lens distortion) for raspberry (currently only IMX219 sensor) cameras. Kept as its own repository so it can be pulled into
mission repos as a git submodule and updated independently of the code.

## Contents

| Path | Resolution | Notes |
|---|---|---|
| `imx219/ost.yaml` | 1280×720 | fx ≈ 554, fy ≈ 560 |
| `imx219_1640x1232/ost.yaml` | 1640×1232 | 2×2-binned, full-FoV sensor mode |
| `imx219_3280x2464/ost.yaml` | 3280×2464 | full sensor; default calibration of `camera_host_server.py` |
| `fisheye_cam.tar.gz` | 1280×720 | raw output of a ROS calibration session: 44 frames (`left-*.png`) plus its own `ost.yaml` / `ost.txt` |

The files come from **separate calibration sessions** and are not rescaled copies of
one another 

## Format

Every `ost.yaml` is the standard ROS `camera_calibration` output:

- `image_width`, `image_height` — resolution the calibration was made at
- `camera_matrix` — 3×3 intrinsics **K**
- `distortion_model: plumb_bob` with `distortion_coefficients` `[k1, k2, p1, p2, k3]`
- `rectification_matrix`, `projection_matrix`

It can be read with `camera_calibration_parsers` / `camera_info_manager` or as plain
YAML. `camera_name: narrow_stereo` is just the calibrator's default name; it only
matters to tools that check it (e.g. `camera_info_manager` logs a mismatch warning).

### Using a calibration at a different resolution

**K** can be scaled linearly to another output resolution **only if the field of view
is the same** (sensor binning or ISP downscaling of the same frame). A cropped sensor
mode (e.g. the IMX219 640×480 / 1920×1080 center crops) sees a different part of the
sensor, so it needs its own calibration. Distortion coefficients are defined in
normalized coordinates and do not change with scaling.

## Use in ROS

The repo should be mounted at `<package>/scripts/camera_correction`:

```bash
# in the superproject (relative URL follows the protocol you cloned with, SSH or HTTPS)
git submodule add ../camera_correction.git <package>/scripts/camera_correction
git submodule update --init --recursive
```

`camera_host_server.py` loads `camera_correction/imx219_3280x2464/ost.yaml` (relative
to the script) by default; override it with the `~calib_file` parameter. The driver
scales **K** to the stream's `~width`×`~height` and uses the calibration for
`~undistort` and for publishing `/camera/camera_info`.

To pick up new calibration data in the superproject:

```bash
cd <package>/scripts/camera_correction && git pull
cd - && git add <package>/scripts/camera_correction && git commit -m "bump camera_correction"
```

## Adding a new calibration 

```bash
rosrun camera_calibration cameracalibrator.py \
    --size <cols>x<rows> --square <square_side_m> \
    --no-service-check \
    image:=/camera/image_raw camera:=/camera
```

`--no-service-check` is needed because the camera driver does not provide a
`set_camera_info` service. After **Calibrate → Save**, the calibrator writes
`/tmp/calibrationdata.tar.gz`. Extract its `ost.yaml` into a new folder named
`<sensor>_<width>x<height>/ost.yaml`, and keep the tarball if you want the raw frames
for re-calibration. Calibrate at the sensor mode and FoV the drone actually streams.

`fisheye_cam.tar.gz` is ~18 MB, below GitHub's 100 MB per-file limit, so Git LFS is
not required.
