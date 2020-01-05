# Overview

This tool is responsible for performing **map matching** operations, by taking raw, noisy GPS readings
and snapping them to the nearest road. Without this functionality, some vehicles might appear to
move off their designated roads.

The program leverages the OSRM map matching REST API, and requires an OSRM instance to be available,
as explained in [map-matcher-osrm-backend](https://github.com/roataway/map-matcher-osrm-backend).

Its second function is to **smoothen direction** data. Like the geographical coordinates, the raw
direction of motion data are noisy, which leads to vehicle markers rotating erratically on the map.
A smoothing algorithm will average the raw values, or perhaps remove outliers, such that the arrow
markers point to a correct direction.

## Architecture
This program combines an MQTT and HTTP service, it uses these resources, given in the configuration:
- `<t_incoming>` - MQTT topic for incoming telemetry, we subscribe to it
- `<t_matched>` - MQTT topic to which matched coordinates are published
- `<osrm>` - the OSRM server, responding to HTTP requests

The pseuso-code of the matcher:

- subscribe to `<t_incoming>` to get raw telemetry from vehicles
- for each incoming MQTT payload
  - update internal state, to keep track of the last `N=10` data points from each vehicle in a queue
  - if we already have `N` points for a given vehicle:
    - send an HTTP request to `<osrm>` with these last points
    - if the response arrived within `M=2` seconds
      - publish the *matched* coordinate to `<t_matched>`
    - otherwise, if we have a timeout
      - just send the raw coordinates to `<t_matched>`
      - discard the response, whenever it arrives
    - apply direction smoothing
  - otherwise just keep accumulating data points

Because the program has to react to independent events that occur at unpredictable rates, it should
be asynchronous. In case of Python, it can use `Twisted` or the new `async/await` syntax; if another
language is used, it should support paradigms that are suitable for this scenario.
