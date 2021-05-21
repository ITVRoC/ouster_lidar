# OS-1 Example Code
Sample code for connecting to and configuring the OS-1, reading and visualizing
data, and interfacing with ROS.

This package allows to change Ouster timestamp mode as ROS parameter and awaits the PTP sync between the Ouster clock and ROS time to start publishing on topics.

### How to compile this?

Compile using **Release** build type to improve performance. If the **Release** mode is not set, the ROS node will have problems for publishing LiDAR messages with a quality greater than 512x10.

```
catkin build ouster_client --cmake-args -DCMAKE_BUILD_TYPE=Release
catkin build ouster_viz --cmake-args -DCMAKE_BUILD_TYPE=Release
catkin build ouster_ros --cmake-args -DCMAKE_BUILD_TYPE=Release
```

### Time synchronizatiom between the Ouster lidar and the host computer

The Precision Time Protocol (PTP) is a protocol used to synchronize clocks in a network. When used in conjunction with hardware support, PTP is capable of sub-microsecond accuracy, which is far better than is normally obtainable with other types of network time sincronization such as NTP.

The Ouster OS1 has the option of PTP time synchronization with the host by setting the `timestamp_mode` to `TIME_FROM_PTP_1588`.

For PTP synchronization two linux programs are used:

- The **ptp4l** program implements the PTP boundary clock and ordinary clock. With hardware time stamping, it is used to synchronize the PTP hardware clock to the master clock, and with software time stamping it synchronizes the system clock to the master clock. 
- The **phc2sys** program is needed only with hardware time stamping, for synchronizing the system clock to the PTP hardware clock on the network interface card (NIC). If no

** For more reading, please visit [Fedora PTP Config Guide](https://jfearn.fedorapeople.org/fdocs/en-US/Fedora_Draft_Documentation/0.1/html/System_Administrators_Guide/ch-Configuring_PTP_Using_ptp4l.html)

### PTP Configuration

This configuration was tested on a Ouster OS1 gen1 (A16) with Ubuntu 16.04 and ROS Kinetic.

#### Check if the card has hardware timestamping available

The result of the ethtool command `sudo ethtool -T eno1` should be similar to the following one:

```
$ sudo ethtool -T eno1
Time stamping parameters for eno1:
Capabilities:
	hardware-transmit     (SOF_TIMESTAMPING_TX_HARDWARE)
	software-transmit     (SOF_TIMESTAMPING_TX_SOFTWARE)
	hardware-receive      (SOF_TIMESTAMPING_RX_HARDWARE)
	software-receive      (SOF_TIMESTAMPING_RX_SOFTWARE)
	software-system-clock (SOF_TIMESTAMPING_SOFTWARE)
	hardware-raw-clock    (SOF_TIMESTAMPING_RAW_HARDWARE)
PTP Hardware Clock: 0
Hardware Transmit Timestamp Modes:
	off                   (HWTSTAMP_TX_OFF)
	on                    (HWTSTAMP_TX_ON)
Hardware Receive Filter Modes:
	none                  (HWTSTAMP_FILTER_NONE)
	all                   (HWTSTAMP_FILTER_ALL)
	ptpv1-l4-sync         (HWTSTAMP_FILTER_PTP_V1_L4_SYNC)
	ptpv1-l4-delay-req    (HWTSTAMP_FILTER_PTP_V1_L4_DELAY_REQ)
	ptpv2-l4-sync         (HWTSTAMP_FILTER_PTP_V2_L4_SYNC)
	ptpv2-l4-delay-req    (HWTSTAMP_FILTER_PTP_V2_L4_DELAY_REQ)
	ptpv2-l2-sync         (HWTSTAMP_FILTER_PTP_V2_L2_SYNC)
	ptpv2-l2-delay-req    (HWTSTAMP_FILTER_PTP_V2_L2_DELAY_REQ)
	ptpv2-event           (HWTSTAMP_FILTER_PTP_V2_EVENT)
	ptpv2-sync            (HWTSTAMP_FILTER_PTP_V2_SYNC)
	ptpv2-delay-req       (HWTSTAMP_FILTER_PTP_V2_DELAY_REQ)
```

#### ptp4l configuration

The ptp4l program implements the PTP boundary clock and ordinary clock

To use [PTP](https://endruntechnologies.com/pdf/PTP-1588.pdf), install with:
```
sudo apt install linuxptp jq
```

Edit the config file `/etc/linuxptp/ptp4l.conf`:

```
sudo vim /etc/linuxptp/ptp4l.conf
```

And set:

```
#clockClass     248
clockClass      128
```

Edit the parameters for starting the service:
```
sudo vim /lib/systemd/system/ptp4l.service

```
Set, where `eno1` is the interface used for PTP.

```
[Unit]
Description=Precision Time Protocol (PTP) service

[Service]
Type=simple
ExecStart=/usr/sbin/ptp4l -f /etc/linuxptp/ptp4l.conf -i eno1

[Install]
WantedBy=multi-user.target
```
  
Activate the service:
```
sudo systemctl daemon-reload
sudo systemctl enable ptp4l
sudo systemctl restart ptp4l
sudo systemctl status ptp4l
```

#### phc2sys configuration

This service will synchronize the system clock to the PTP hardware clock on the network interface card

Edit the service file: 

```
sudo vim /lib/systemd/system/phc2sys.service
```

Set, where `eno1` is the interface used for PTP. The `-O 0` parameter is to overcome a leap second offset (now its 36 seconds).

```
[Unit]
Description=Synchronize system clock or PTP hardware clock (PHC)
After=ntpdate.service
Requires=ptp4l.service
After=ptp4l.service

[Service]
Type=simple
ExecStart=/usr/sbin/phc2sys -w -s eno1 -O 0

[Install]
WantedBy=multi-user.target
```

Activate the service:
```
sudo systemctl daemon-reload
sudo systemctl enable phc2sys
sudo systemctl restart phc2sys
sudo systemctl status phc2sys
```


### Troubleshooting

#### Message time delay

The delay must be less than 1:

`rostopic delay /os1_cloud_node/imu`

#### Local hardware clock verification

To verify if the hardware clock is synchronized with the computer run 

`sudo phc_ctl eno1 get | grep -Po '[0-9\.]{10,30}' | xargs -I {} date -d @{} && date`

The two dates must be equal:

```
espeleo@espeleo-robo:~$ sudo phc_ctl eno1 get | grep -Po '[0-9\.]{10,30}' | xargs -I {} date -d @{} && date
Sex Set 25 18:04:46 -03 2020
Sex Set 25 18:04:46 -03 2020
```

#### Remote LiDAR clock verification

Acces the LiDAR web api time information using `wget` command. First install the `jq` command for visualizing the response in a clear way.

`sudo apt install jq`

Verify system clock:

`wget -nv -O- http://192.168.1.61/api/v1/system/time/system | jq`

Example:

```
{
  "tracking": {
    "remote_host": "ptp",
    "last_offset": -1.0602e-05,
    "frequency": 11.014,
    "skew": 1.913,
    "root_delay": 1e-09,
    "rms_offset": 9.603e-06,
    "reference_id": "70747000",
    "system_time_offset": 3.09e-06,
    "residual_frequency": -0.016,
    "ref_time_utc": 1601068094.6893108,
    "leap_status": "normal",
    "root_dispersion": 0.000119325,
    "update_interval": 2,
    "stratum": 1
  },
  "monotonic": 1995.410123256,
  "realtime": 1601068097.5325494
}

```

Verify PTP configuration:

`wget -nv -O- http://192.168.1.61/api/v1/system/time/ptp | jq`


### Package contents

See the `README.md` in each subdirectory for details.

#### Contents
* [ouster_client/](ouster_client/README.md) contains an example C++ client for the OS-1 sensor
* [ouster_viz/](ouster_viz/README.md) contains a visualizer for the OS-1 sensor
* [ouster_ros/](ouster_ros/README.md) contains example ROS nodes for publishing point cloud messages

#### Sample Data
* Sample sensor output usable with the provided ROS code is available
  [here](https://data.ouster.io/sample-data-1.12)
