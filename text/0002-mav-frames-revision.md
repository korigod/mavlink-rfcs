* Start date: 2018-03-15
* Contributors: Andrei Korigodski <akorigod@gmail.com>, Oleg Kalachev <okalachev@gmail.com>
* RFC PR: [mavlink/rfcs#2](https://github.com/mavlink/rfcs/pull/2)
* Related issues: [mavlink/mavlink#861](https://github.com/mavlink/mavlink/issues/861)

# Summary

Revise MAVLink local frames of reference to create a clear and comprehensive frames system. Agree on frames naming convention, add some useful frames and correct descriptions.

# Motivation

Rich enough but clear set of coordinate frames is essential for effective drone control. Current frame set is quite obscure, often leads to confusion and lacks some useful frames.

Let's check current non-global coordinate frames and their description:

* `MAV_FRAME_LOCAL_NED` — Local coordinate frame, Z-up (x: north, y: east, z: down).
* `MAV_FRAME_LOCAL_ENU` — Local coordinate frame, Z-down (x: east, y: north, z: up)
* `MAV_FRAME_LOCAL_OFFSET_NED` — Offset to the current local frame. Anything expressed in this frame should be added to the current local frame position.
* `MAV_FRAME_BODY_NED` — Setpoint in body NED frame. This makes sense if all position control is externalized - e.g. useful to command 2 m/s^2 acceleration to the right.
* `MAV_FRAME_BODY_OFFSET_NED` — Offset in body NED frame. This makes sense if adding setpoints to the current flight path, to avoid an obstacle - e.g. useful to command 2 m/s^2 acceleration to the east.

Meaning of the first two frames, `MAV_FRAME_LOCAL_NED` and `MAV_FRAME_LOCAL_ENU`, is obvious. The next three are much less clear. First of all, they are defined as "setpoints" or "offsets" instead of frames of reference.

Both `_OFFSET_` frames are not supported by PX4 Firmware at all, while `MAV_FRAME_BODY_NED` is supported only for velocity setpoints (where it acts as Front-Right-Down oriented frame, not NED).

Also sometimes it's necessary to have a continuous position, without discrete jumps. In ROS it's called `odom`.

# Detailed Design

Proposed frame naming convention is:

`MAV_FRAME_origin_rotation[_modifier]`,

where `origin` can be:

* `LOCAL` for frames with local origin fixed with respect to Earth
* `BODY` for frames with origin fixed to the vehicle
* `BODY_TERRAIN` for frames with origin with is projection of `BODY` origin on underlying terrain (i.e. altitude estimated using sonar or laser rangefinder)

`rotation` can be:

* `NED` for North-East-Down
* `ENU` for East-North-Up
* `FRD` for Forward-Right-Down, where Forward and Right are tangential to Earth and Down is normal to it
* `FLU` for Forward-Left-Up, where Forward and Left are tangential to Earth and Up is normal to it
* `RPY` for Roll-Pitch-Yaw (fixed in orientation to the moving vehicle, z axis points down when vehicle is level, right-handed)
* `PRY` for Pitch-Roll-Yaw (fixed in orientation to the moving vehicle, z axis points up when vehicle is level, right-handed)

and `modifier` can be:

* `ODOM`

__________________________________________________________

Proposed non-global frames are:

* `MAV_FRAME_LOCAL_NED`
* `MAV_FRAME_LOCAL_NED_ODOM`
* `MAV_FRAME_LOCAL_ENU`
* `MAV_FRAME_LOCAL_FRD`
* `MAV_FRAME_BODY_NED`
* `MAV_FRAME_BODY_FRD`
* `MAV_FRAME_BODY_RPY`

## Implementation

To implement proposed changes `MAV_FRAME` enum will be extended with following entries:

* `MAV_FRAME_LOCAL_NED_ODOM`
* `MAV_FRAME_LOCAL_FRD`
* `MAV_FRAME_BODY_FRD`
* `MAV_FRAME_BODY_RPY`

and decriptions of following will be updated:

* `MAV_FRAME_LOCAL_OFFSET_NED` and `MAV_FRAME_BODY_OFFSET_NED` will be marked deprecated
* `MAV_FRAME_BODY_NED` description will be aligned with proposed meaning

# Alternatives

* Drop `LOCAL` for frames with zero in local origin (proposal by [hamishwillee](https://github.com/hamishwillee) (?))
* Use `HOR` or `HORIZ` instead of `FRD`
* Use `BODY` instead of `RPY` to describe frames with axes tied to vehicle body. In this case to describe frames with origin in vehicle IMU (?) `OFFSET` or other symbol can be used
* Assign large int values (like 50+) to all local `MAV_FRAME`s to prevent mixing of global and local frames in docs and source code

# Drawbacks

* Significantly extending `MAV_FRAME` enum
* Changing of frame naming convention and meaning of existing frame `MAV_FRAME_BODY_NED` can cause confusion
* `RPY` and `PRY` look similar and could be unintentionally mixed up

# Prior art

## Large aircraft common frames

Common non-global frames are:

* Vehicle-carried NED (corresponds to `MAV_FRAME_LOCAL_NED`)
* Velocity-oriented, or wind (with x axis aligned with vehicle velocity)
* Body (corresponds to `MAV_FRAME_BODY_RPY`)

## ROS frames convention

## MAVROS frames

* `map` (corresponds to `MAV_FRAME_LOCAL_ENU`)
* `base_link` (corresponds to `MAV_FRAME_BODY_PRY`)
* `fcu_frd` (corresponds to `MAV_FRAME_BODY_RPY`)

## CLEVER drone kit frames

CLEVER is open source PX4-compatible platform based on ROS ([documentation in Russian](https://copterexpress.gitbooks.io/clever/content/docs/frames.html)). It supports following frames of reference:

* `local_origin` (corresponds to `MAV_FRAME_LOCAL_ENU`)
* `fcu_horiz` (corresponds to `MAV_FRAME_LOCAL_FLU`)
* `fcu` (corresponds to `MAV_FRAME_LOCAL_PRY`)

ENU is used instead of NED to comply with ROS guidelines.

## DJI Mobile SDK frames

DJI Mobile SDK supports two frames of reference ([[1]](https://developer.dji.com/mobile-sdk/documentation/introduction/flightController_concepts.html#coordinate-systems), [[2]](https://developer.dji.com/mobile-sdk/documentation/introduction/component-guide-flightController.html)):

* `Ground` (corresponds to `MAV_FRAME_LOCAL_NED`)
* `Body` (corresponds to `MAV_FRAME_BODY_FRD`)

Yaw angle is always relative to North.

# Unresolved Questions

* `FRD` and `RPY` naming
* `BODY_NED` meaning will be changed
* Necessity of `MAV_FRAME_LOCAL_FRD` is not obvious

# References

[REP-105](http://www.ros.org/reps/rep-0105.html)

http://www.perfectlogic.com/articles/avionics/flightdynamics/flightpart2.html

https://www.mathworks.com/help/aeroblks/about-aerospace-coordinate-systems.html
