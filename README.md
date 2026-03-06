# How to test with ROS2

Docs: https://github.com/bondada-a/erobs/blob/humble-experimental/docs/Container_documentation.md


# Env vars

NOTE: This is optional, was not needed when we were testing on Linux.

```bash
export ROS_DOMAIN_ID=0
```

# Docker setup

Start the server:
```bash
podman run --arch=amd64 -it --rm --network host --ipc=host --pid=host ghcr.io/bondada-a/beambot_img:latest bash
```

Inside the podman container:

```bash
source /root/ws/erobs/install/setup.bash && ros2 launch beambot beambot_bringup.launch.py use_fake_hardware:=true enable_vision:=false
```

Client side on the server image:
```bash
ros2 action send_goal /beambot_execution beambot_interfaces/action/MTCExecution "{full_json: '$(cat /root/ws/erobs/src/cms/tasks/beamtime/spincoat_to_hotplate.json)'}"
```

# Conda/pixi setup

```bash
pixi run ros2 topic list
pixi run ros2 node list
```

# Build beambot_interfaces (pixi)

```bash
git clone https://github.com/bondada-a/erobs.git -b humble-experimental
cd erobs
rm -rf build/beambot_interfaces install/beambot_interfaces
colcon build --packages-select beambot_interfaces \
  --cmake-args \
  -DPython3_EXECUTABLE=$(which python3) \
  -DPYTHON_EXECUTABLE=$(which python3) \
  -DPYTHON_SOABI=cpython-311-x86_64-linux-gnu
```

# Send goal via IPython

```bash
export FASTRTPS_DEFAULT_PROFILES_FILE=/tmp/fastrtps_no_shm.xml
export PYTHONPATH=$PWD/src:$PYTHONPATH
source install/setup.bash
ipython
```

```python
import rclpy
rclpy.init()
from bluesky_ros.mtc_ophyd_device import MTCExecutionDevice
from bluesky import RunEngine, plans as bps

RE = RunEngine({})
robot = MTCExecutionDevice()
RE(bps.abs_set(robot, 'src/cms/tasks/beamtime/spincoat_to_hotplate.json'))
```

# FastRTPS config (required to be able to send commands to ros server)

Create `/tmp/fastrtps_no_shm.xml` — disables shared memory transport to avoid `/dev/shm` permission issues:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<profiles xmlns="http://www.eprosima.com/XMLSchemas/fastRTPS_Profiles">
    <transport_descriptors>
        <transport_descriptor>
            <transport_id>udp_transport</transport_id>
            <type>UDPv4</type>
        </transport_descriptor>
    </transport_descriptors>
    <participant profile_name="default_participant" is_default_profile="true">
        <rtps>
            <useBuiltinTransports>false</useBuiltinTransports>
            <userTransports>
                <transport_id>udp_transport</transport_id>
            </userTransports>
        </rtps>
    </participant>
</profiles>
```


