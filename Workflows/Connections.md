---
id: c1bs7wsjfbhb0zipaywqv1
title: Connection Rework
desc: ""
updated: 1765425402022
created: 1654223767390
currentStep: 0
totalSteps: 0
---
# Connections are love, Connections are life.

Currently, many aspects of the flowsheet UI can be considered functional, but less than ideal. This presents a bottleneck in broader access to platform capabilities as the flowsheet is the core interface through which models are defined and broader insights are accessed. There are several areas wherein our current workflows can be improved.
## Connection path generation
Connections are one of the most obvious areas where the existing implementation is poor. This is largely due to the fact that the original code for generating connection paths was done to achieve a functional output that was necessary to demonstrate flowsheeting capability, but improving the QoL aspects of managing connections was not a high priority compared to other tasks at the time. Since this original implementation, there have been a few refactors to the connection path generation logic, however these refactors were centred around improving the implementation as it already was, rather than changing the way in which connections more broadly behave. In my view, most of the issues with the current status quo approach stem from one fundamental aspect of the implementation: **Connection paths are rendered entirely within the frontend**. 

In practice, this means that paths are rendered in an on demand fashion with each rerender of the canvas. The general logic for this is as follows:
1. Information about the graphic objects for each operation/stream are parsed to find their position, size, rotation, and flipped status.
1. The information about each respective connection is parsed to reveal the direction (inlet/outlet), the operation it is connected to, and the id of the respective port.
1. This information is sent to the *ConnectionGraphic* function, which then identifies the points for the connection path by passing the start/end op and connection data to the *getConnectionLinePoints* method, which has the following logic
    1. It calculates the local coordinates for the start and end of the connection path, being the port position for each respective operation/stream.
    1. It finds the notch location for the start and end of the path, being an offset of 10px from the port. This ensures that there is always some separation of the main part of the path and the outline of the operation such that the associated port connection is always visible.
    1. These positions are all adjusted to account for the rotation status of the operation.
    1. It calls the *checkVertical* method to determine the placement of the middle points. If the middle of the connection path is vertical, then two points will be added with the same x position, halfway between that of the start and end notch locations, and the same y position as its respective notch. If it is not vertical, then the middle points will have the same x position as their respective notch, with a y position halfway between the y of each notch. 
1. These points are then used to render the line

Essentially, there are 5 segments rendered for each connection line, *Sometimes it looks like only 3 as some segments share x or y positions*. One from the outlet of the start op to the first notch, one from the notch to the middle, one between the middle (being vertical or horizontal), one from the middle to the end notch, one from the end notch to the inlet of the end op.

### So, why is this bad?
Because paths are generated ondemand, it is not possible for users to edit them. It is also very difficult to identify when paths cross. Instead, the only real option a user has is to drag around and rotate operations and streams on the flowsheet with the hope that they might eventually find a layout with a nice path structure.
### How could we solve this?
With some fun new implementations. Rather than calculating all points in a path everytime, we could calculate the points upon the inital creation of the connection. We can store these points in a new model table (I know, gross, but bear with me). Moving operatations at either end of the connection would update these points as required in a similar fashion to how we handle unit operation movement (ie fully cached during movement with an upate upon being dropped). Movement of operations would update the start/end points and their associated notch for the operation in question, as well as the x/y position of the middle segment point on for the same side and/or its vertical status. We should give the user the ability to select and shift the middle segment. In the case of a vertical middle segment, the user could adjust its x positioning. For horizontal middle segments, the user could adjust y positions. This is similar to how such behaviour is achieved in diagrams.net, as shown in the below figures:

>![Horizontal Segment](/assets/images/hLineShift.png)
---
>![Vertical Segment](/Workflows/assets/images/vLineShift.png)

As an additional option, we could also potentially give them the ability to toggle the vertical nature of the middle connection as well, but that's just a bonus extra...

This approach could be a nice way to achieve more user customisation and better general flowsheet layouts without requiring a massive change to our existing implementation or worrying about more challenging features such as adding additional intermediate path points. It might also be useful for achieving better visualisation of path intersections, as we could write some functions to identify potentially intersecting paths based on path coordinates, allowing us to then in turn visualise such cases with standard approaches such as hops or breaks.

![](/assets/images/conArc.png)
---
![](/assets/images/conGap.png)


## Unit operation rotation
Currently, rotating operations also rotates the connected stream nodes in some instances, messing up the pre-existing layout. This is disgusting and should not happen.
![](/assets/images/nonRotated.png)
---
![](/assets/images/rotated.png)

## Connection port positioning and ordering
The status quo implementation of connection ordering is very limited.
- It is not possible to reorder connections
- Inlets and Outlets can only be placed on one side.

A possible solution to this is as follows:
- We could have a way of docking the start/end of a connection to available points around the edge of the operation in question. This is shamelessly stolen from diagrams.net
- Each side could have a set number of possible connection docking points, with number and positioning increasing/decreasing as connections are added in operations such as mixers/splitters

![](/assets/images/connectionDocking.png)