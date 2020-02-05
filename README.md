# OS-1 Example Code
Sample code for connecting to and configuring the OS-1, reading and visualizing
data, and interfacing with ROS.

This package allows to change Ouster timestamp mode as ROS parameter and awaits the PTP sync between the Ouster clock and ROS time to start publishing on topics.
### PTP

To use [PTP](https://endruntechnologies.com/pdf/PTP-1588.pdf), install with:
```
sudo apt install linuxptp
```

Edit the parameters for starting the service:
```
sudo vim /lib/systemd/system/ptp4l.service

```
Set:

```
ExecStart=/usr/sbin/ptp4l -i <your ethernet interface> -S
```
  
Start the service:
```
service ptp4l start
```

Make it run on boot:
```
sudo systemctl enable ptp4l
```

See the `README.md` in each subdirectory for details.

## Contents
* [ouster_client/](ouster_client/README.md) contains an example C++ client for the OS-1 sensor
* [ouster_viz/](ouster_viz/README.md) contains a visualizer for the OS-1 sensor
* [ouster_ros/](ouster_ros/README.md) contains example ROS nodes for publishing point cloud messages

## Sample Data
* Sample sensor output usable with the provided ROS code is available
  [here](https://data.ouster.io/sample-data-1.12)
