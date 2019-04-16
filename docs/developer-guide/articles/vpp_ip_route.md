# Control Flow Diagrams

## Example: Route waiting for the associated interface

This example shows how route, received from KVDB before the associated
interface, gets delayed and stays in the `PENDING` state until the interface
configuration is received and applied. This is achieved simply by listing the
key representing the interface among the dependencies for the route.


![CFD](../../img/control-flow-diagram/add_route_before_interface.svg?sanitize=true)

[<< Back to control flow diagram index](../kvscheduler.md#index)