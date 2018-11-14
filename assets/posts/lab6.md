Lab 6
=====

## Overview and Motivations - Charlotte

Lab 5 gave us the ability to localize the car on a given map, Stata basement tunnels in this case. It is finally time for us now to learn how to drive and control. Lab 6 is consisted of two main components, path planning and trajectory control. The safety controller implemented in Lab 3 is also integrated to the car to prevent from crashing into obstacles. At the end of this lab, we are able to drive the car around Stata basement tunnels autonomously with specified start and end poses.

## Proposed Approach - Charlotte

We decided to split the lab half and work in parallel. Half of the team implemented a pure pursuit controller for trajectory control so that the car could precisely drive along a predefined path in autonomous mode. The other half of the team focused on the development of path planning. After comparing different algorithms’ pros and cons, we decided to use RRT and RRT* to plan and optimize the car’s trajectory on a given map of Stata basement tunnels with specified start and goal poses. Finally, we integrated the implementations of both parts and tested on the car in real life.

### Initial Setup - Michelle
We made two nodes for this lab. One was responsible for creating a trajectory using path planning while another one was for controlling the robot to follow the trajectory. We also made an extra tool that allowed us to create our own paths by clicking points in Rviz in order to be able to test the trajectory follower in isolation. 

### Technical Approach - Josh, Charlotte, and Frederic

Trajectory Tracking - Josh

To implement trajectory tracking, we used a pure pursuit controller.  A pure pursuit controller takes in the car’s position and a trajectory as a list of points.  A circle is generated with a radius equal to its lookahead parameter centered at the car.  The intersections between this circle and the lines making up the trajectory are found and stored.  Then, the controller finds the closest point on the trajectory to the car and uses this point to find which of the intersections is next in the trajectory.  Once this goal point is found, the controller finds an arc of a circle that the car is currently tangent to and that passes through the goal point.  The wheelbase of the car is used to convert this arc into a steering angle that will send the car along the arc.  
A crucial parameter in the pure pursuit controller is the lookahead distance.  Too small a lookahead distance and the car may overshoot turns, but too large a lookahead and the car may cut corners.  Both of these increase cross track error, the shortest distance between the car and the trajectory, and can lead to the car hitting obstacles.  The optimal lookahead distance also depends on various factors, including the speed of the car and the shape of the trajectory.  
Small lookahead distances can also be problematic if the car is further than the lookahead distance away from the trajectory.  If this happens, the car is lost as it cannot find a goal point.  To fix this issue, whenever the car cannot find a goal point, we multiply the lookahead distance by 2.  Every time a goal point is found, we divide the lookahead distance by 2 until we once again reach the original lookahead.  In our simulations, the lookahead distance was never multiplied by more than 4, as it took a very small lookahead and the car aggressively overshooting a turn for the car to get that far away from the trajectory.

<iframe width="560" height="315" src="https://www.youtube.com/embed/OihOHUsrgeE" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

_Figure 1: Trajectory Tracking in Real World_



Path Planner - Frederic

In this portion, we will discuss our implementation of RRT* to find a path for the robot to follow from some start to goal location. There are several components which we were able to adjust to improve path planning for this specific scenario. 

First, we deal with the map as an occupancy grid. We map out a numpy array containing either free space or obstacles for every pixel in the map. This allows us to easily dilate obstacles such as walls and doors to ensure that our eventual path does not near these obstacles and corners. 

At a high level, we would like to create a dense tree in the map where edges represent possible paths to follow. To do this, we sample pixel values as potential nodes in the tree. We found that sampling pixels closer to the goal state with higher probability makes our algorithm perform more efficiently, even though this is traded off with eventual path optimality. 

We also use Bresenhams line algorithm to validate that edges in our tree do not run through obstacles in the map. This is a fast algorithm which works well with our numpy arrays. Similarly we use a fast scipy method to calculate the nearest node in the tree to connect the newly sampled point to.

RRT* is very similar to RRT but with an additional cost function. A random point is sampled on the map. Then, we find a point on the existing path that is the closest from the random sampled point. If the random sampled point is within d distance away from the closest point, we call it as the new point, else the new point is defined d distance away from the closest point in random sampled point direction. Unlike RRT which only looks between the new point and the closest point, this simple RRT* algorithm checks whether the paths between the new point and every existed path vertex point are valid (A path is valid when there is no obstacle in between.), and it also compares and sees which existing path vertex has the lowest cost to go from path vertex to the new point. The cost of a path vertex is defined as the distance of the path from the start to that vertex. Therefore, the new point will be appended after an existing path vertex with the lowest cost. This way, the total path distance is minimized and the planned trajectory is optimal. 

<iframe width="560" height="315" src="https://www.youtube.com/embed/VC03Krj4GP0" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

_Figure 2: Path Planning_

To future improve our estimated trajectory, we also implemented a smooth function to avoid excessive unnecessary path vertices. After we obtain a defined path, the smooth function compares to every other vertices and check if they consist of a valid path. For example, it will check the first and the third vertices. If they make a valid path, then the second vertex will be neglected. Therefore, the number of path vertices is minimized, and the the number of turns the car has to perform is minimized. The final trajectory is ideal.      



We optimized two distance variables; one to determine how close we have to get to the goal to conclude the algorithm, and another to decide how far away newly sampled points can be from the tree. After optimizing overall runtime and pure pursuit error, we found that 4 metres and 1.5 metres were optimal for these parameters respectively.

Once we sample a pixel within 1.5 metres of the goal state, we finish updating our algorithm and calculate the path that takes the robot from start to finish. The representation of the path as a PolygonStamped is used to match that which the pure pursuit node uses.



### ROS Implementation - Michelle

This lab was unique because we were provided with minimal amounts of starter code, which gave us a lot of flexibility in how we implemented the program. We structured the lab so that it would be easy to work on the path planner and trajectory follower separately. In order to avoid having to wait for a working path planning before working on the trajectory follower, we made a program that allowed us to manually create custom paths by clicking in Rviz. 
There also wasn’t an autograder, so we had to provide a framework to evaluate the success of the program ourselves. The trajectory follower is able to test a set of lookahead values in succession and store the errors in a text file so we can analyze the data afterwards. 


## Experimental Evaluation - Josh

To test our implementations of pure pursuit and RRT\*, we ran the implementations in simulation and on the robot.  We collected data on values of interest, including elapsed time and cross track error for pure pursuit and time to completion and path length in RRT\*.  These data can be used to optimize moveable parameters in these implementations.

### Testing Procedure - Victor and Josh

Trajectory Tracking - Josh

We tested pure pursuit by having the car follow a pre-planned trajectory around the Stata basement in simulation.  We decided to test in simulation because we felt it would reduce noise immensely while still providing an accurate model of the real world.  Perhaps most importantly, testing in simulation allowed us to know the location of the car perfectly, a crucial element in calculating cross track error.  It also eliminated noise from the hardware, such as having changing battery voltages, and the environment, such as large objects being left in the hall or people walking near the car.  Ultimately, using the simulation allowed us to test pure pursuit in isolation by removing most of the noise. 
For each loop around the basement, we changed the lookahead distance the car used for pure pursuit and measured cross track error and total elapsed time.  Each time the car received its location from the odometry, we calculated the cross track error.  This allowed us to observe cross track error over time for each lookahead value as well as calculate average error over the loop.  We also timed each loop since we care about the fastest traversal of the loop for the final race.


Path Planning -  Victor

We tested our path planner by visualizing the points in the RRT on Rviz. While our RRT sampled points and attempted to add them to the tree, we visualized all of the sampled points on Rviz. Any point that was valid, or was able to be added to the tree, was visualized in green, with a line pointing from the parent point to the new point. Any sampled point that was invalid, either too far away from any other points on the tree or blocked by an obstacle, was visualized in red. 
Our first goal was to make sure all of the points sampled were accurately placed inside the tunnels on the map, and not inside an obstacle. Once all the sampled points were inside the tunnels, this meant that we were correctly transforming from pose to pixel, and pixel to pose correctly, and that we were only sampling from possible valid points. From there we had to make sure that none of the points were marked incorrectly, and that none of the edges connecting points came in contact with an obstacle. We tested this visually in our Rviz representation until we met all of these criteria. 


### Results - Michelle

The main parameter to tune for an optimized trajectory is the lookahead distance. In order to find this optimal value, we used the results from the following graphs. We were looking for a value that would minimize the average error from the planned trajectory and minimize the overall time to complete a loop. 
<center>![](assets/images/lab6/lookahead_error.png =400x300)</center>
<center> _Figure 3 : Average error vs. Lookahead distance_</center> 
Figure 3 shows that at low lookahead values, we get high amounts of error and as the lookahead distance increases, the average error approaches infinity. However, with a lookahead value of around 1.5m, the error is minimized. 

<center>![](assets/images/lab6/lookahead_time.png =400x300)</center>
<center>  _Figure 4 : Basement loop time vs. Lookahead distance_</center>
Figure 4 shows that the time per loop decreased as we increased the lookahead value past 1.2 meters. 

Based on these graphs, we chose 1.5 to be our lookup value since it provided a good trade off of having a fast time while also minimizing error compared to our planned path. Looking ahead to the race next week, we knew it was ultimately important to have a fast loop time. However, if our program is far off from the trajectory that was planned, there is a risk of running into obstacles. 

After optimizing the lookahead value, we evaluated the overall success of the trajectory follower by graphing error over time. 
<center>![](assets/images/lab6/time_error.png =400x300)</center>
<center> _Figure 5 : Error vs. Time_</center>
For a lookahead value of 1.5, we are able to follow the path within millimeters when going straight. When turning, the error stays below 0.5m. Although this value seems high, it is reasonable given that the turns are tight and the robot has a dynamic constraint on how quickly it can turn. This error value for turning could easily be decreased by a more intelligent velocity controller that slows down at the turns. 

## Conclusions - Michelle and Josh

Lessons Learned - Michelle and Josh

This lab emphasized the importance of using debugging outputs and concrete evidence to evaluated the performance of our own code. At the beginning of the lab, we jumped quickly to implement the algorithms without thoroughly creating debugging outputs and testing along the way. As a result, we spent almost double that time afterwards going through each part of the code struggling to find out which part wasn’t working.  However, we did do a much better job this week in effectively dividing tasks.  In past labs, we all kind of working on all pieces of the lab because we were not certain how the whole thing was supposed to work.  This week, we divided the system clearly into modules and figured out beforehand what the expected inputs and outputs to each module were.  This little bit of extra communication at the beginning allowed us to work easily in parallel.  We communicated well throughout the lab and effectively helped each other meet our goals.  Additionally, we were not afraid to communicate constructively with one another to ensure the team continnued moving forward.
We also learned the importance of quantitatively testing our implementations.  In past labs, we often just ran our algorithms, and agreed they seemed to "work".  This lab, we worked hard to quantify how well our algorithms "worked".  We gathered data on how long our implementations took to run and how far they were from the optimal solution.  This allowed us to tune our parameters in a more justifiable manner.  We weren't just arguing that this parameter value "looked the best", we could actually argue that this parameter minimized error.

Future Work - Josh

While our trajectory tracking and path planning worked, there is still room for optimization.  For pure pursuit, we could reduce the speed of the car as the car starts turning to minimize error going around turns.  For path planning, our algorithm could certainly be sped up.  It quickly finds early points in the path and then slows down while finding the final few points.  We could even run these later calculations while the car is driving.  The car doesn't need to know the whole path at the start, it just needs to know the few beginning segments.  By the time the car nears the end of the path, our path planning algorithm will have completed the path with high probability so the car will be able to finish the loop.  This could cut down immensely on the amount of time our car sits stationary while planning the route.

