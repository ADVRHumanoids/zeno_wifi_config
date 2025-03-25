# Zenoh for ROS2 WiFi Communication

<p align="center">
  <img src="https://zenoh.io/img/zenoh-dragon.svg" height="150">
</p>

[![License](https://img.shields.io/badge/License-EPL%202.0-blue.svg)](https://www.eclipse.org/legal/epl-2.0/)
[![Discussion](https://img.shields.io/badge/discussion-on%20discourse-ff69b4.svg)](https://discourse.ros.org/t/zenoh-bridge-ros2-dds-lightweight-communication-for-real-robots/30388)

## Overview

This guide helps you configure Zenoh to optimize ROS2 communications over WiFi networks. Zenoh serves as a more efficient alternative to standard DDS for wireless robot communication, reducing bandwidth usage and improving reliability.

### Why Zenoh for ROS2 WiFi?

- **Bandwidth Efficiency**: Zenoh uses a more compact protocol than DDS, reducing network traffic
- **Improved Reliability**: Better handling of intermittent connectivity in wireless environments
- **Simplified Multi-Robot Setup**: Namespace configuration at the bridge level instead of per-node
- **Fine-Grained Control**: Manage topic frequency, prioritization, and filtering
- **Lower Latency**: Optimized for wireless environments with minimal overhead

## Installation

### Option 1: Debian/Ubuntu (Recommended)

```bash
# Add Eclipse Zenoh repository
echo "deb [trusted=yes] https://download.eclipse.org/zenoh/debian-repo/ /" | sudo tee -a /etc/apt/sources.list > /dev/null
sudo apt update

# Install standalone bridge
sudo apt install zenoh-bridge-ros2dds

# Or install plugin for existing Zenoh router
sudo apt install zenoh-plugin-ros2dds
```

### Option 2: Docker

```bash
docker pull eclipse/zenoh-bridge-ros2dds:latest
```

## Basic Configuration

Zenoh bridges use JSON5 configuration files. Here's the basic structure:

```json5
{
  "plugins": {
    "ros2dds": {
      "nodename": "zenoh_bridge_ros2dds",
      "namespace": "/robot_name"
    }
  },
  "mode": "router",
  "listen": {
    "endpoints": ["tcp/0.0.0.0:7447"]
  },
  "connect": {
    "endpoints": ["tcp/192.168.1.100:7447"]
  }
}
```

## Configuration Examples

### Robot-Side Configuration

```json5
{
  "plugins": {
    "ros2dds": {
      "nodename": "zenoh_bridge_robot",
      "namespace": "/robot1",
      "domain": 0,
      "pub_max_frequencies": [
        ".*/camera/image_raw=5",
        ".*/lidar_scan=10"
      ]
    }
  },
  "mode": "router",
  "listen": {
    "endpoints": ["tcp/0.0.0.0:7447"]
  }
}
```

### Lab/Control Station Configuration

```json5
{
  "plugins": {
    "ros2dds": {
      "nodename": "zenoh_bridge_lab",
      "namespace": "/"
    }
  },
  "mode": "router",
  "connect": {
      "tcp/<ROBOT_IP_ADDRESS>:7447"
  }
}
```

### Multi-Robot Configuration

```json5
{
  "plugins": {
    "ros2dds": {
      "nodename": "zenoh_bridge_central",
      "namespace": "/"
    }
  },
  "mode": "router",
  "connect": {
    "endpoints": [
      "tcp/192.168.1.101:7447",  // Robot 1
      "tcp/192.168.1.102:7447",  // Robot 2
      "tcp/192.168.1.103:7447"   // Robot 3
    ]
  }
}
```

## Advanced Configuration Options

### Namespace Management

The `namespace` parameter is critical for multi-robot setups:

- **Robot Bridge**: Set to a unique identifier (e.g., `/robot1`, `/jazzy`)
  - Topics within the robot (like `/camera/image`) will automatically be prefixed when seen by the lab (`/robot1/camera/image`)
  - When the lab sends to `/robot1/cmd_vel`, the robot sees it as just `/cmd_vel`

- **Lab Bridge**: Usually set to root namespace (`/`)
  - Allows interaction with all robots without adding extra prefixes
  - Makes multi-robot visualization and monitoring easier

```json5
"namespace": "/robot1"  // For robot bridge
"namespace": "/"        // For lab bridge
```

### Topic Filtering

Control which topics traverse the network using `allow` or `deny` lists:

```json5
"allow": {
  "publishers": [".*/laser_scan", "/tf", ".*/pose"],
  "subscribers": [".*/cmd_vel"],
  "service_servers": [".*/.*_parameters"],
  "action_servers": [".*/navigate_to_pose"]
}
```

```json5
"deny": {
  "publishers": ["/rosout", "/parameter_events", ".*/raw_image"],
  "subscribers": ["/rosout"]
}
```

You cannot use both `allow` and `deny` in the same configuration. The best approach is:
- Use `allow` when you want to limit traffic to a small set of known topics
- Use `deny` when you want to block just a few problematic topics

### Frequency Control

Limit how often high-bandwidth topics cross the network:

```json5
"pub_max_frequencies": [
  ".*/camera/image_raw=5",      // Limit to 5 Hz
  ".*/lidar_scan=10",           // Limit to 10 Hz
  "/tf=20",                     // Limit to 20 Hz
  "/battery_status=0.1"         // Once every 10 seconds
]
```

Frequency control should be applied at the source of the data (typically on the robot bridge), not at the destination.

### Message Prioritization

Ensure critical messages get through even during network congestion:

```json5
"pub_priorities": [
  "/emergency_stop=1:express",  // Highest priority, immediate delivery
  "/cmd_vel=2:express",         // High priority, immediate delivery
  "/pose=3",                    // Medium-high priority
  "/diagnostic=7"               // Lowest priority
]
```

Priority values range from 1 (highest) to 7 (lowest), with 5 as default. The `express` option bypasses batching for lower latency.

### Network Configuration Options

```json5
// Configure Zenoh to operate as a router
"mode": "router",

// Listen for incoming connections
"listen": {
  "endpoints": ["tcp/0.0.0.0:7447"]
},

// Connect to other bridges/routers
"connect": {
  "endpoints": ["tcp/192.168.1.100:7447"]
},

// Enable auto-discovery (for more dynamic setups)
"scouting": {
  "multicast": {
    "enabled": true,
    "autoconnect": { "router": ["router"] }
  }
}
```

## Running Zenoh Bridge

Run the Zenoh bridge with your configuration file:

```bash
zenoh-bridge-ros2dds -c your_config.json5
```

To prevent DDS traffic from going over WiFi, set:

```bash
# For ROS2 Iron and later:
export ROS_AUTOMATIC_DISCOVERY_RANGE=LOCALHOST


## Testing Your Configuration

To verify bridges are communicating properly:

```bash
# List topics from all robots
ros2 topic list

# Publish a test message
ros2 topic pub /test_topic std_msgs/msg/String "{data: 'Jazzy robot is online'}" --rate 1

# Subscribe to a topic with the robot's namespace
ros2 topic echo "/namespace"/test_topic
```

## Troubleshooting

### Common Issues

1. **Bridges not connecting**:
   - Verify IP addresses and port numbers
   - Check for firewalls blocking port 7447
   - Ensure both bridges are running

2. **Topics not appearing**:
   - Check namespace configuration
   - Verify topics aren't filtered by `allow`/`deny` settings
   - Confirm ROS nodes are publishing

3. **Port conflicts when testing locally**:
   - Use different ports for each bridge:
     ```json
     "listen": {
       "endpoints": ["tcp/127.0.0.1:7448"]
     }
     ```

## Best Practices

1. **Limit high-bandwidth topics**: Use `pub_max_frequencies` to reduce network load from cameras, lidars, and other sensors.

2. **Prioritize control messages**: Give motion and safety commands the highest priority with `pub_priorities`.

3. **Configure namespaces carefully**: Use unique namespaces for each robot, and root namespace for the lab.

4. **Use localhost isolation**: Set `ROS_LOCALHOST_ONLY=1` to prevent DDS from communicating directly over WiFi.

5. **Start simple, then optimize**: Begin with basic connectivity, then fine-tune with frequency controls and filtering.

6. **Monitor bandwidth usage**: Use network monitoring tools to identify bandwidth-heavy topics that need additional controls.

## Additional Resources

- [Zenoh Documentation](https://zenoh.io/docs/)
- [Zenoh-ROS2-DDS GitHub Repository](https://github.com/eclipse-zenoh/zenoh-plugin-ros2dds)
- [ROS2 Middleware Report on Zenoh](https://discourse.ros.org/t/ros-2-alternative-middleware-report/)
