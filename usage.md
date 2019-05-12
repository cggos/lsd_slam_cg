# 3 Usage
LSD-SLAM is split into two ROS packages, `lsd_slam_core` and `lsd_slam_viewer`. `lsd_slam_core` contains the full SLAM system, whereas `lsd_slam_viewer` is optionally used for 3D visualization.
Please also read **General Notes for good results** below.

## 3.1 `lsd_slam_core`
We provide two different usage modes, one meant for live-operation (`live_slam`) using ROS input/output, and one `dataset_slam` to use on datasets in the form of image files.

### 3.1.1 Using `live_slam`
If you want to directly use a camera.

    rosrun lsd_slam_core live_slam /image:=<yourstreamtopic> /camera_info:=<yourcamera_infotopic>

When using ROS camera_info, only the image dimensions and the `K` matrix from the camera info messages will be used - hence the video has to be rectified.

Alternatively, you can specify a calibration file using

    rosrun lsd_slam_core live_slam /image:=<yourstreamtopic> _calib:=<calibration_file>

In this case, the camera_info topic is ignored, and images may also be radially distorted. See the Camera Calibration section for details on the calibration file format.


### 3.1.2  Using `dataset_slam`

    rosrun lsd_slam_core dataset_slam _files:=<files> _hz:=<hz> _calib:=<calibration_file>

Here, `<files>` can either be a folder containing image files (which will be sorted alphabetically), or a text file containing one image file per line. `<hz>` is the framerate at which the images are processed, and `<calibration_file>` the camera calibration file.

Specify `_hz:=0` to enable sequential tracking and mapping, i.e. make sure that every frame is mapped properly. Note that while this typically will give best results, it can be much slower than real-time operation.


### 3.1.3 Camera Calibration
LSD-SLAM operates on a pinhole camera model, however we give the option to undistort images before they are being used. You can find some sample calib files in `lsd_slam_core/calib`.

#### Calibration File for FOV camera model:

    fx/width fy/height cx/width cy/height d
    in_width in_height
    "crop" / "full" / "none" / "e1 e2 e3 e4 0"
    out_width out_height


Here, the values in the first line are the camera intrinsics and radial distortion parameter as given by the PTAM cameracalibrator, in\_width and in\_height is the input image size, and out\_width out\_height is the desired undistorted image size. The latter can be chosen freely, however 640x480 is recommended as explained in section 3.1.6. The third line specifies how the image is distorted, either by specifying a desired camera matrix in the same format as the first four intrinsic parameters, or by specifying "crop", which crops the image to maximal size while including only valid image pixels.


#### Calibration File for Pre-Rectified Images
This one is without radial distortion correction, as a special case of ATAN camera model but without the computational cost:

    fx/width fy/height cx/width cy/height 0
    width height
    none
    width height


#### Calibration File for OpenCV camera model:

    fx fy cx cy k1 k2 p1 p2
    inputWidth inputHeight
    "crop" / "full" / "none" / "e1 e2 e3 e4 0"
    outputWidth outputHeight


### 3.1.4 Useful Hotkeys

- `r`: Do a full reset

- `d / e`: Cycle through debug displays (in particular color-coded variance and color-coded inverse depth).

- `o`: Toggle on screen info display

- `m`: Save current state of the map (depth & variance) as images to `lsd_slam_core/save/`

- `p`: Brute-Force-Try to find new constraints. May improve the map by finding more constraints, but will block mapping for a while.

- `l`: Manually indicate that tracking is lost: will stop tracking and mapping, and start the re-localizer.



### 3.1.5 Parameters (Dynamic Reconfigure)
A number of things can be changed dynamically, using (for ROS fuerte)

    rosrun dynamic_reconfigure reconfigure_gui

or (for ROS indigo)

    rosrun rqt_reconfigure rqt_reconfigure

Parameters are split into two parts, ones that enable / disable various sorts of debug output in `/LSD_SLAM/Debug`, and ones that affect the actual algorithm, in `/LSD_SLAM`.
Note that debug output options from `/LSD_SLAM/Debug` only work if lsd\_slam\_core is built with debug info, e.g. with `set(ROS_BUILD_TYPE RelWithDebInfo)`.

* `minUseGrad`: [double] Minimal absolute image gradient for a pixel to be used at all. Increase if your camera has large image noise, decrease if you have low image-noise and want to also exploit small gradients.
* `cameraPixelNoise`: [double] Image intensity noise used for e.g. tracking weight calculation. Should be set larger than the actual sensor-noise, to also account for noise originating from discretization / linear interpolation.
* `KFUsageWeight`: [double] Determines how often keyframes are taken, depending on the overlap to the current keyframe. Larger -> more keyframes.
* `KFDistWeight`: [double] Determines how often keyframes are taken, depending on the distance to the current Keyframe. Larger -> more keyframes.
* `doSLAM`: [bool] Toggle global mapping component on/off. Only takes effect after a reset.
* `doKFReActivation`: [bool] Toggle keyframe re-activation on/off: If close to an existing keyframe, re-activate it instead of creating a new one. If false, the map will continually grow even if the camera moves in a relatively constrained area; If false, the number of keyframes will not grow arbitrarily.
* `doMapping`: [bool] Toggle entire keyframe creating / update module on/off: If false, only tracking stays active, which will prevent rapid motion or moving objects from corrupting the map.
* `useFabMap`: [bool] Use openFABMAP to find large loop-closures. Only takes effect after a reset, and requires LSD-SLAM to be compiled with FabMap.
* `allowNegativeIdepths`: [bool] Allow idepth to be (slightly) negative to avoid introducing a bias for far-away points.
* `useSubpixelStereo`: [bool] Compute subpixel-accurate stereo disparity.
* `useAffineLightningEstimation`: [bool] EXPERIMENTAL: Correct for global affine intensity changes during tracking. Might help if you have problems with auto-exposure.
* `multiThreading`: [bool] Toggle multi-threading of depth map estimation. Disable for less CPU usage, but possibly slightly less quality.
* `maxLoopClosureCandidates`: [int] Maximal number of loop-closures that are tracked initially for each new keyframe.
* `loopclosureStrictness`: [double] Threshold on reciprocal loop-closure consistency check, to be added to the map. Larger -> more (possibly wrong) loop-closures.
* `relocalizationTH`: [double] How good a relocalization-attempt has to be to be accepted. Larger -> more strict.
* `depthSmoothingFactor`: [double] How much to smooth the depth map. Larger -> less smoothing.


Useful for debug output are:

* `plotStereoImages`: [bool] Plot searched stereo lines, and color-coded stereo-results. Nice visualization of what's going on, however drastically decreases mapping speed.
* `plotTracking`: [bool] Plot final tracking residual. Nice visualization of what's going on, however drastically decreases tracking speed.
* `continuousPCOutput`: [bool] Publish current keyframe's point cloud after each update, to be seen in the viewer. Nice visualization, however bad for performance and bandwidth.







### 3.1.6 General Notes for Good Results

* Use a **global shutter** camera. Using a rolling shutter will lead to inferior results.
* Use a lens with a **wide field-of-view** (we use a 130° fisheye lens).
* Use a **high framerate**, at least 30fps (depending on the movements speed of course). For our experiments, we used between 30 and 60 fps.
* We recommend an image resolution of **640x480**, significantly higher or lower resolutions may require some hard-coded parameters to be adapted.
* LSD-SLAM is a monocular SLAM system, and as such cannot estimate the absolute scale of the map. Further it requires **sufficient camera translation**: Rotating the camera without translating it at the same time will not work. Generally sideways motion is best - depending on the field of view of your camera, forwards / backwards motion is equally good. Rotation around the optical axis does not cause any problems.
* During initialization, it is best to move the camera in a circle parallel to the image without rotating it. The scene should contain sufficient structure (intensity gradient at different depths).
* **Adjust** `minUseGrad` **and** `cameraPixelNoise` to fit the sensor-noise and intensity contrast of your camera.
* If tracking / mapping quality is poor, try decreasing the keyframe thresholds `KFUsageWeight` and `KFDistWeight` slightly to generate more keyframes.
* Note that LSD-SLAM is very much non-deterministic, i.e. results will be different each time you run it on the same dataset. This is due to parallelism, and the fact that small changes regarding when keyframes are taken will have a huge impact on everything that follows afterwards.


## 3.2 LSD-SLAM Viewer
The viewer is only for visualization. It can also be used to output a generated point cloud as .ply.
For live operation, start it using

    rosrun lsd_slam_viewer viewer

You can use rosbag to record and re-play the output generated by certain trajectories. Record & playback using

    rosbag record /lsd_slam/graph /lsd_slam/keyframes /lsd_slam/liveframes -o file_pc.bag
    rosbag play file_pc.bag

You should never have to restart the viewer node, it resets the graph automatically.

If you just want to lead a certain pointcloud from a .bag file into the viewer, you
can directly do that using

    rosrun lsd_slam_viewer viewer file_pc.bag



### 3.2.1 Useful Hotkeys

- `r`: Reset, will clear all displayed data.

- `w`: Print the number of points / currently displayed points / keyframes / constraints to the console.

- `p`: Write currently displayed points as point cloud to file lsd_slam_viewer/pc.ply, which can be opened e.g. in meshlab. Use in combination with sparsityFactor to reduce the number of points.


### 3.2.2 Parameters (Dynamic Reconfigure)

- `showKFCameras `: Toggle drawing of blue keyframe camera-frustrums. min: False, default: True, max: True
- `showKFPointclouds `: Toggle drawing of point clouds for all keyframes. min: False, default: True, max: True
- `showConstraints `: Toggle drawing of red/green pose-graph constraints. min: False, default: True, max: True
- `showCurrentCamera `: Toggle drawing of red frustrum for the current camera pose. min: False, default: True, max: True
- `showCurrentPointcloud `: Toggle drawing of the latest point cloud added to the map. min: False, default: True, max: True
- `pointTesselation `: Size of points. min: 0.0, default: 1.0, max: 5.0
- `lineTesselation `: Width of lines. min: 0.0, default: 1.0, max: 5.0
- `scaledDepthVarTH `: log10 of threshold on point's variance, in the respective keyframe's scale. min: -10.0, default: -3.0, max: 1.0
- `absDepthVarTH `: log10 of threshold on point's variance, in absolute scale. min: -10.0, default: -1.0, max: 1.0
- `minNearSupport `: Only plot points that have #minNearSupport similar neighbours (higher values remove outliers). min: 0, default: 7, max: 9
- `cutFirstNKf `: Do not display the first #cutFirstNKf keyframe's point clouds, to remove artifacts left-over from the random initialization. min: 0, default: 5, max: 100
- `sparsifyFactor `: Only plot one out of #sparsifyFactor points, selected at random. Use this to significantly speed up rendering for large maps. min: 1, default: 1, max: 100
- `sceneRadius `: Defines near- and far clipping plane. Decrease to be able to zoom in more. min: 1, default: 80, max: 200
- `saveAllVideo `: Save all rendered images... only use if you know what you are doing. min: False, default: False, max: True
- `keepInMemory `: If set to false, the point cloud is only stored in OpenGL buffers, and not kept in RAM. This greatly reduces the required RAM for large maps, however also prohibits saving / dynamically changing sparsifyFactor and variance-thresholds. min: False, default: True, max: True