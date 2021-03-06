# Try SROS2 in MacOS 

## Install OpenSSL

Install a recent version of OpenSSL (1.0.2g or later)

```
brew install openssl
```

You will need to have OpenSSL on your library path to run DDS-Security demos, you can do this by running:
```
export DYLD_LIBRARY_PATH=`brew --prefix openssl`/lib:$DYLD_LIBRARY_PATH
```
For convenience you can add this export to your bash_profile.

## Install ROS2

### Install from binaries 

First install ROS2 from binaries following [these instructions](https://github.com/ros2/ros2/wiki/OSX-Install-Binary)


Setup your environment:
```bash
source . ~/ros2_install/ros2-osx/setup.bash
``` 

In the rest of these instructions we assume that every terminal setup the environment as instructed above.


### Install from source

For OpenSSL to be found when building code you'll need to define this environment variable:
```bash
export OPENSSL_ROOT_DIR=`brew --prefix openssl`
```
For convenience you can add this export to your bash_profile.

Install ROS2 from source following [these instructions](https://github.com/ros2/ros2/wiki/Linux-Development-Setup)

Note: Fast-RTPS requires an additional CMake flag to build the security plugins so the ament invocation needs to be modified to pass:
```bash
src/ament/ament_tools/scripts/ament.py build --build-tests --symlink-install --cmake-args -DSECURITY=ON --
```

Setup your environment:
```bash
source ~/ros2_ws/install/setup.bash
``` 

In the rest of these instructions we assume that every terminal setup the environment as instructed above.

## Preparing the environment for the demo

### Create a folder for the files required by this demo

We will now create a folder to store all the files necessary for this demo:

```bash
mkdir ~/sros2_demo
```

### Generating a keystore, keys and certificates

#### Generate a keystore

```bash
cd ~/sros2_demo
ros2 security create_keystore demo_keys
```

#### Generate keys and certificates for the talker and listener nodes

```bash
ros2 security create_key demo_keys talker
ros2 security create_key demo_keys listener
```

### Define the SROS2 environment variables

```bash
export ROS_SECURITY_ROOT_DIRECTORY=$(pwd)/demo_keys
export ROS_SECURITY_ENABLE=true
export ROS_SECURITY_STRATEGY=Enforce
```

These variables need to be defined in each terminal used for the demo. For convenience you can add it to your bash_profile.

## Run the demo

ROS2 allows you to [change DDS implementation at runtime](https://github.com/ros2/ros2/wiki/Working-with-multiple-RMW-implementations).
This demo can be run with fastrtps by setting:
```bash
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```
And with Connext by setting:
```bash
export RMW_IMPLEMENTATION=rmw_connext_cpp
```

### Authentication and Encryption

Run the `talker` demo program:

```bash
ros2 run demo_nodes_cpp talker
```

In another terminal (after preparing the terminal as previously described), we will do the same thing with the `listener` program:
```bash
ros2 run demo_nodes_py listener
```

Note: You can switch between the C++ and Python packages arbitrarily.


At this point, your `talker` and `listener` nodes should be communicating securely, using explicit access control lists!
Hooray!

### Access Control (RTI Connext only, from source only)

The previous demo used authentication and encryption, but not access control, which means that any authenticated node would be able to publish and subscribe to any data stream (aka topic).
To increase the level of security in the system, you can define strict limits, known as access control, which restrict what each node is able to do.
For example, one node would be able to publish to a particular topic, and another node might be able to subscribe to that topic.
To do this, we will use the sample policy file provided in `examples/sample_policy.yaml`.

First, we will copy this sample policy file into our keystore:

```bash
curl -sk https://raw.githubusercontent.com/ros2/sros2/ardent/examples/sample_policy.yaml -o ./demo_keys/policies.yaml
```

And now we will use it to generate the XML permission files expected by the middleware:

```bash
ros2 security create_permission demo_keys talker demo_keys/policies.yaml
ros2 security create_permission demo_keys listener demo_keys/policies.yaml
```

Then, in one terminal (after preparing the terminal as previously described), run the `talker` demo program:

```
RMW_IMPLEMENTATION=rmw_connext_cpp ros2 run demo_nodes_cpp talker
```

In another terminal (after preparing the terminal as previously described), we will do the same thing with the `listener` program:

```
RMW_IMPLEMENTATION=rmw_connext_cpp ros2 run demo_nodes_py listener
```

