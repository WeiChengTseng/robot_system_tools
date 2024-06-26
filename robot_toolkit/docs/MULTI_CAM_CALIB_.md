<!-- # Documentation and steps on how to use Kalibr for multicamera calibration -->
# Multicamera calibration step-by-step introduction

## What you need to prepare
- camera, such as Realsense D415
- 


## Installation
### Docker Engine
There are several ways to install Docker on your computer. If you already install docker on your computer, please skip this part.
You can check [this](https://docs.docker.com/engine/install/ubuntu/) out.

1. Set up Docker's apt repository.
    ```bash
    # Add Docker's official GPG key:
    
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    # Add the repository to Apt sources:
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update
    ```
2. Install the Docker packages.
    ```bash
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```

3. Verify that the Docker Engine installation is successful by running the hello-world image.
    ```bash
    sudo docker run hello-world
    ```
    This command downloads a test image and runs it in a container. When the container runs, it prints a confirmation message and exits.

### Kalibr


2. build docker container for Kalibr


```bash
source devel/setup.bash
rosrun kalibr kalibr_calibrate_cameras --bag /data/ball_dropping/calibration.bag --target /data/3x5.yaml --models pinhole-radtan pinhole-radtan pinhole-radtan  --topics /cam_1/color/image_raw /cam_2/color/image_raw /cam_3/color/image_raw
```

## Introduction

[Kalibr](https://github.com/ethz-asl/kalibr) is a tool box that can be used to calibrate visual-inertial systems, i.e cameras and IMUs. We use it primarily for calibrating multiple cameras' intrinsics and the extrinsics between the cameras, below you will find the steps to achieve the same. 

Kalibr provides a docker container that includes all the dependancies and creates the ros workspace that is build the source code. They have a really nice [wiki](https://github.com/ethz-asl/kalibr/wiki), which was a great reference to create this documentation. You can think of this documentation of cherry picked instructions from the wiki and extra information that is not provided or is unclear in the wiki. 

The high level process of how kalibr should be used is:
- clone the reposiory and build the docker container 
- create a "data" folder that will be mounted to the container 
- create a calibration tag using the tool they provide [here](https://github.com/ethz-asl/kalibr/wiki/calibration-targets)
- Collect a ROSbag of the cameras' "image_raw" topics that will be used by Kalibr to compute the intrinsics and extrinsics 
- Create a .yaml file that describes the calibration tag used. 
- Both the bags and the .yaml should be present in the "data folder"
- run Kalibr calibration 
    - each topic in the ROS bag are processed to estimate the 2D keypoints of the checkerboard or apriltag corners 
    - These keypoints are used to estimate an initial guess for the intrinsics based on the camera model you pass as the argument. 
    - These keypoints are also used to estimate an initial guess of the extrinsics based on epipolar geometry 
    - The initial intrinsics and extrinsics guesses are then used to solve a batch optimization problem that minimizes reprojection error of the detected Keypoints keeping the cameras' intrinsics and extrinsics as the design variables.
- You can find more detailed description of these steps below- 

## Usage 

1. First clone kalibr repository and build the docker container they provide 

```bash
mkdir $USER/git 
cd $HOME/git 
git clone https://github.com/ethz-asl/kalibr
cd $HOME/git/kalibr

docker build -t kalibr -f Dockerfile_ros1_20_04 . # change this to whatever ubuntu version you want
```
2. run the docker container with:

```bash
xhost +local:root
xhost +local:docker
FOLDER=/home/ruthrash/utm/data
sudo docker run -it -e "DISPLAY" -e "QT_X11_NO_MITSHM=1" \
    -v "/tmp/.X11-unix:/tmp/.X11-unix:rw" \
    -v "$FOLDER:/data" kalibr
```
where FOLDER is the location where we would store our rosbags. **make sure** it points to the right path in your local machine

3. Now, use the kalibr_create_target_pdf node to create a calibration target as described [here](https://github.com/ethz-asl/kalibr/wiki/calibration-targets)

- We suggest using Aprilgrid over the checkerboard as their points can be detected even under partial occlusion of tags 
- We also suggest to create a calibration tag as big as possible with maximum possible dimensions of the april tag marker and separation between them. 
- an example command would be: 
```bash
source devel/setup.bash
rosrun kalibr kalibr_create_target_pdf --nx 3 --ny 3 --tsize 0.045 --tspace 0.2 --type apriltag
```
follow instructions [here](https://github.com/ethz-asl/kalibr/wiki/calibration-targets) to figure out tspace and tsize

4. Once you have collected the data, you can run the calibration node like so, 
```bash
source devel/setup.bash
rosrun kalibr kalibr_calibrate_cameras --bag /data/ball_dropping/calibration.bag --target /data/3x5.yaml --models pinhole-radtan pinhole-radtan pinhole-radtan  --topics /cam_1/color/image_raw /cam_2/color/image_raw /cam_3/color/image_raw
```
Here we are calibrating 3 cameras with pinhole-radtan model. **make sure** to modify the command for your case
