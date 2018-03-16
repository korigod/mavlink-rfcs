* Start date: 2018-03-15
* Contributors: Andrei Korigodski <akorigod@gmail.com>
* Related issues: https://github.com/mavlink/mavlink/issues/861

# Summary

The scope of this document is to revision MAVLink frames of reference. The goal is to create a clear and comprehensive frames system.

# Motivation

Rich enough but clear set of coordinate frames is essential for effective drone control. Current frame set is quite obscure, often leads to confusion and lacks some useful frames.

Let's check current non-global coordinate frames and their description:

* **MAV_FRAME_LOCAL_NED.** Local coordinate frame, Z-up (x: north, y: east, z: down).
* **MAV_FRAME_LOCAL_ENU.** Local coordinate frame, Z-down (x: east, y: north, z: up)
* **MAV_FRAME_LOCAL_OFFSET_NED.** Offset to the current local frame. Anything expressed in this frame should be added to the current local frame position.
* **MAV_FRAME_BODY_NED.** Setpoint in body NED frame. This makes sense if all position control is externalized - e.g. useful to command 2 m/s^2 acceleration to the right.
* **MAV_FRAME_BODY_OFFSET_NED.** Offset in body NED frame. This makes sense if adding setpoints to the current flight path, to avoid an obstacle - e.g. useful to command 2 m/s^2 acceleration to the east.

Also sometimes it's necessary to have a continuous position, without discrete jumps. In ROS it's `odom`.

# Detailed Design

Proposed frame naming convention is:

`MAV_FRAME_origin_rotation[_modifier]`,

where `origin` can be:

* `LOCAL` for frames with origin in local_origin
* `BODY` for frames with origin in vehicle center of mass

`rotation` can be:

* `NED` for North-East-Down
* `ENU` for East-North-Up
* `FRD` for Forward-Right-Down, where Forward and Right are tangential to Earth and Down is normal to it
* `LFU` for Left-Forward-Up, where Left and Forward are tangential to Earth and Up is normal to it
* `RPY` for Roll-Pitch-Yaw (right-handed)
* `PRY` for Pitch-Roll-Yaw (right-handed)

and `modifier` can be:

* `ODOM`

__________________________________________________________

Proposed non-global frames are:

* MAV_FRAME_LOCAL_NED
* MAV_FRAME_LOCAL_NED_ODOM
* MAV_FRAME_LOCAL_ENU
* MAV_FRAME_BODY_NED
* MAV_FRAME_BODY_FRD
* MAV_FRAME_BODY_RPY

# Alternatives

* Drop `LOCAL` for frames with zero in local_origin

# Prior art

## Large aircraft common frames

## MAVROS frames

* map (corresponds to MAV_FRAME_LOCAL_ENU)
* base_link (corresponds to )
* fcu (ENU-like MAV_FRAME_BODY_RPY variant)
* fcu_frd (corresponds to MAV_FRAME_BODY_RPY)

## DJI Mobile SDK frames

[docs](https://developer.dji.com/mobile-sdk/documentation/introduction/flightController_concepts.html#coordinate-systems)
[docs](https://developer.dji.com/mobile-sdk/documentation/introduction/component-guide-flightController.html)

* Ground (corresponds to MAV_FRAME_LOCAL_NED)
* Body (corresponds to MAV_FRAME_BODY_FRD)

Yaw angle is always relative to North.

## CLEVER drone kit frames

CLEVER drone kit is open source PX4-compatible platform based on ROS [(documentation in Russian)](https://copterexpress.gitbooks.io/clever/content/docs/frames.html). It supports following frames of reference:

* local_origin (corresponds to MAV_FRAME_LOCAL_ENU)
* fcu_horiz (ENU-like MAV_FRAME_BODY_FRD variant)
* fcu (ENU-like MAV_FRAME_BODY_RPY variant)

ENU is used instead of NED to comply with ROS guidelines.

# Adoption strategy

# Unresolved Questions

* FRD and RPY naming.
* BODY_NED meaning will be changed.

# References

[REP-105](http://www.ros.org/reps/rep-0105.html)
