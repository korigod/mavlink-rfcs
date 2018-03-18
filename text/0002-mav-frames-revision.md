* Start date: 2018-03-15
* Contributors: Andrei Korigodski <akorigod@gmail.com>, Oleg Kalachev <okalachev@gmail.com>
* RFC PR: [mavlink/rfcs#2](https://github.com/mavlink/rfcs/pull/2)
* Related issues: [mavlink/mavlink#861](https://github.com/mavlink/mavlink/issues/861)

# Summary

Revise MAVLink local frames of reference to create a clear and comprehensive frames system. Agree on frames naming convention, add some useful frames and correct descriptions.

# Motivation

Rich enough but clear set of coordinate frames is essential for effective drone control. Current frame set is quite obscure, lacks some useful frames and often leads to confusion ([[1]](https://github.com/ArduPilot/ardupilot/issues/2447), [[2]](https://github.com/ArduPilot/ardupilot/issues/4717), [[3]](https://groups.google.com/forum/#!topic/px4users/X9rclmM9AUw)).

Let's check current non-global coordinate frames and their description:

* `MAV_FRAME_LOCAL_NED` — Local coordinate frame, Z-up (x: north, y: east, z: down).
* `MAV_FRAME_LOCAL_ENU` — Local coordinate frame, Z-down (x: east, y: north, z: up)
* `MAV_FRAME_LOCAL_OFFSET_NED` — Offset to the current local frame. Anything expressed in this frame should be added to the current local frame position.
* `MAV_FRAME_BODY_NED` — Setpoint in body NED frame. This makes sense if all position control is externalized - e.g. useful to command 2 m/s^2 acceleration to the right.
* `MAV_FRAME_BODY_OFFSET_NED` — Offset in body NED frame. This makes sense if adding setpoints to the current flight path, to avoid an obstacle - e.g. useful to command 2 m/s^2 acceleration to the east.

Meaning of the first two frames, `MAV_FRAME_LOCAL_NED` and `MAV_FRAME_LOCAL_ENU`, is obvious. The next three are much less clear. First of all, they are defined as "setpoints" or "offsets" instead of frames of reference.

`MAV_FRAME_BODY_NED` description makes clear that it's frame with body orientation (fixed to vehicle): `e.g. useful to command 2 m/s^2 acceleration to the right`, however it's name implies it's NED frame, which is confusing. Description of `MAV_FRAME_BODY_OFFSET_NED` makes thing even more complicated because it looks like it's a real NED frame: `e.g. useful to command 2 m/s^2 acceleration to the east`.

The only detailed explanation of current local frames' meaning is in [APM documentation](http://ardupilot.org/dev/docs/copter-commands-in-guided-mode.html#set-position-target-local-ned). It describes `MAV_FRAME_LOCAL_OFFSET_NED` as vehicle-carried NED and `MAV_FRAME_BODY_OFFSET_NED` as vehicle-carried Front-Right-Down frame which is clear. However, behavior of `MAV_FRAME_BODY_NED` frame is complicated:

_Positions are relative to the vehicle’s home position in the North, East, Down (NED) frame. Use this to specify a position x metres north, y metres east and (-) z metres above the home position. Velocity directions are relative to the current vehicle heading. Use this to specify the speed forward, right and down (or the opposite if you use negative values)._

So the behavior of frames for positions and for velocities is not always the same, which is confusing and should be avoided. Naming pattern is unclear:

* `BODY` frames are not always aligned with respect to vehicle orientation;
* `NED` doesn't really mean anything except direction of z axis as e.g. `MAV_FRAME_BODY_OFFSET_NED` is not NED-oriented.

Both `_OFFSET_` frames are not supported by PX4 firmware at all, while `MAV_FRAME_BODY_NED` is supported only for velocity setpoints (where it acts as Front-Right-Down oriented frame, not NED).

Also sometimes it's necessary to have a continuous position, without discrete jumps, like integrated vehicle velocity, which is always quite correctly represents vehicle movement in short-term although can drift in long-term. In ROS such a frame is called `odom`.

# Detailed Design

Proposed frame naming convention is:

`MAV_FRAME_origin_rotation[_modifier]`,

where `origin` can be:

* `LOCAL` for frames with local origin fixed with respect to Earth;
* `BODY` for frames with origin fixed to the vehicle;
* `BODY_TERRAIN` for frames with origin with is projection of `BODY` origin on underlying terrain (i.e. altitude estimated using sonar or laser rangefinder),

`rotation` can be:

* `NED` for North-East-Down;
* `ENU` for East-North-Up;
* `FRD` for Forward-Right-Down, where Forward and Right are tangential to Earth and Down is normal to it;
* `FLU` for Forward-Left-Up, where Forward and Left are tangential to Earth and Up is normal to it;
* `RPY` for Roll-Pitch-Yaw (fixed in orientation to the moving vehicle, z axis points down when vehicle is leveled, right-handed);
* `PRY` for Pitch-Roll-Yaw (fixed in orientation to the moving vehicle, z axis points up when vehicle is leveled, right-handed),

and `modifier` can be:

* `ODOM`.

__________________________________________________________

Proposed non-global frames are:

* `MAV_FRAME_LOCAL_NED`;
* `MAV_FRAME_LOCAL_NED_ODOM`;
* `MAV_FRAME_LOCAL_ENU`;
* `MAV_FRAME_LOCAL_FRD`;
* `MAV_FRAME_BODY_NED`;
* `MAV_FRAME_BODY_FRD`;
* `MAV_FRAME_BODY_RPY`.

## Implementation

To implement proposed changes `MAV_FRAME` enum will be extended with following entries:

* `MAV_FRAME_LOCAL_NED_ODOM`;
* `MAV_FRAME_LOCAL_FRD`;
* `MAV_FRAME_BODY_FRD`;
* `MAV_FRAME_BODY_RPY`;

and decriptions will be updated as stated:

* `MAV_FRAME_LOCAL_OFFSET_NED` and `MAV_FRAME_BODY_OFFSET_NED` will be marked deprecated;
* `MAV_FRAME_BODY_NED` description will be aligned with proposed meaning.

# Alternatives

* Drop `LOCAL` for frames with zero in local origin (proposal by [hamishwillee](https://github.com/hamishwillee) (?)).
* Use `HOR` or `HORIZ` instead of `FRD`.
* Use `BODY` instead of `RPY` to describe frames with axes tied to vehicle body. In this case to describe frames with origin fixed to vehicle `OFFSET` or other symbol can be used. However, there is still need for ENU-like alternative.
* Assign large int values (like 50+) to all local `MAV_FRAME`s to prevent mixing of global and local frames in docs and source code.

# Drawbacks

* Changes will require corresponding amends of APM and PX4 firmwares and documentation, possibly also some other software which is using MAVLink. However, everything will still work as there will not be introduced any breaking changes in type enumerations. New developers can be confused if frames will behave differently than MAVLink descriptions tell but in fact they are confused now as well. APM docs can be changed simultaneously with the code (and therefore the firmware behavior) so it's not a major problem and it seems like other platform just don't have any docs on this matter.
* `MAV_FRAME` type enumeration will be extended significantly.
* Naming convention and meaning of existing frame `MAV_FRAME_BODY_NED` will be changed which can cause confusion.
* `RPY` and `PRY` look similar and could be unintentionally mixed up.

# Prior art

## Large aircraft common frames

Common non-global frames are ([[1]](http://www.perfectlogic.com/articles/avionics/flightdynamics/flightpart2.html), [[2]](https://www.mathworks.com/help/aeroblks/about-aerospace-coordinate-systems.html)):

* Vehicle-carried NED (corresponds to `MAV_FRAME_LOCAL_NED`);
* Velocity-oriented, or wind (with x axis aligned with vehicle velocity);
* Body (corresponds to `MAV_FRAME_BODY_RPY`).

## ROS frames convention

[REP-105](http://www.ros.org/reps/rep-0105.html) specifies naming conventions and semantic meaning for coordinate frames of mobile platforms used with ROS. The basic frames are:

* `map` (corresponds to `MAV_FRAME_LOCAL_ENU`);
* `odom` (corresponds to `MAV_FRAME_LOCAL_ENU_ODOM`);
* `base_link` (corresponds to `MAV_FRAME_BODY_PRY`).

## MAVROS frames

* `map` (corresponds to `MAV_FRAME_LOCAL_ENU`);
* `base_link` (corresponds to `MAV_FRAME_BODY_PRY`);
* `fcu_frd` (corresponds to `MAV_FRAME_BODY_RPY`).

## ardrone_autonomy frames

Ardrone_autonomy is ROS package for AR.drone control. It [provides](http://ardrone-autonomy.readthedocs.io/en/latest/frames.html) following frames of reference:

* `odom` (corresponds to `MAV_FRAME_LOCAL_ENU_ODOM`);
* `ardrone_base_link` (corresponds to `MAV_FRAME_BODY_PRY`);
* `ardrone_base_frontcam`;
* `ardrone_base_bottomcam`.

## CLEVER drone kit frames

[CLEVER](https://github.com/CopterExpress/clever) is open source PX4-compatible platform based on ROS. It supports following frames of reference:

* `local_origin` (corresponds to `MAV_FRAME_LOCAL_ENU`);
* `fcu_horiz` (corresponds to `MAV_FRAME_LOCAL_FLU`);
* `fcu` (corresponds to `MAV_FRAME_LOCAL_PRY`).

ENU is used instead of NED to comply with ROS guidelines.

## DJI Mobile SDK frames

DJI Mobile SDK supports two frames of reference ([[1]](https://developer.dji.com/mobile-sdk/documentation/introduction/flightController_concepts.html#coordinate-systems), [[2]](https://developer.dji.com/mobile-sdk/documentation/introduction/component-guide-flightController.html)):

* `Ground` (corresponds to `MAV_FRAME_LOCAL_NED`);
* `Body` (corresponds to `MAV_FRAME_BODY_FRD`).

Yaw angle is always relative to north.

# Unresolved Questions

* It's not obvious are `FRD` and `RPY` designations clear enough.
* Necessity of `MAV_FRAME_LOCAL_FRD` is not obvious.
* Necessity of `ODOM` frame is not obvious.
* Necessity of some frame with `BODY_TERRAIN` origin.
* On some platforms yaw is specified w.r.t. north even in frames with orientation fixed to the vehicle (like `Body` frame of DJI SDK). It would be possibly better to explicitly prohibit this behavior as it leads to inconsistent frame meaning.
