Final Challenge - The Labyrinth
=====

## Overview and Motivations - Charlotte, Josh

Throughout the course of 6.141, we have learned about the use of different sensors on the car for it to autonomously navigate in space, such as using Lidar to localize the car on a given map and using the camera to identify the target objects. We have also implemented several different path planning algorithms to predetermine an optimal path for the car as well as a trajectory following algorithm to ensure the car stay on its path. For our final challenge, we chose to do The Labyrinth. Its objective is to autonomously escape a non-perfect maze of narrow pathways, with no prior knowledge other than an estimated size of the maze, as fast as possible. 

Escaping from an unknown labyrinth poses a fundamentally new set of challenges from racing around the Stata basement.  The basic building blocks, path planning and pure pursuit, are the same but most be optimized for a very different environment.


## Proposed Approach - Charlotte

The challenge is divided into three components as the team were divided in half to work in parallel. Half of the team is responsible for setting up Google Cartographer as well as tuning the trajectory following algorithm, including the pure pursuit, three-point turn controller, and safety controller. The other half of the team takes on the task to implement a path planning algorithm first on a static map, and then on a dynamic map. Lastly, the whole team will work together and integrate both parts and test it in a real life environment. 


### Initial Setup - Charlotte

There are a few things we know about the maze that could work as initial conditions. First, we know that the shape of the maze is undefined yet bounded by a known radius. Second, the maze is single leveled, multicursal (branching), and not simply connected (loops). Third, the walls in the maze are opaque and matte, therefore, no information can be gain beyond the wall via Lidar and camera. Lastly, the exit and the relative location of the starting point to the maze are both unknown.       

Google Cartographer is used to provide real-time localization and a 2D SLAM map. Rapidly-exploring Random Tree (RRT) is implemented for fast and optimal dynamic path planning.  


### Technical Approach

#### Exiting the Maze - Ricky

Our high level strategy for exiting the maze is as follows; first, we pick some goal state outside of the maze and plan a path to it. As the robot pursues this path and goal, it updates its known environment. As the robot’s certainty of free space and obstacles grows, and it transpires that the current plan is obstacle ridden, the robot dynamically replans its path to another goal state. This procedure is repeated until the robot successfully arrives at a goal state, which by definition is outside of the maze. 

Our approach to path planning is based on Rapidly-exploring Rapid Tree (RRT). The path planner takes as input a map or occupancy grid and an estimated pose from the SLAM cartographer. 

We sample points in and around the maze and build up a tree of valid edges for RRT to use. We assume that unknown and unexplored territory is open space for the robot until the robot moves into those areas.

Once the tree has expanded outside of the maze radius, the path is published for pure pursuit to consume. Our algorithm constantly validates the current path as we may find that the given path is impeded by an obstacle. If an obstacle invalidates the path, instead of reproducing the tree, we cut off only the invalidated subtree using Breadth First Search. We are able to do this since each node in the tree holds pointers to both its parents and children.

The tree is expanded again to another goal state, and a new path is formed dynamically. For now, the robot will be stationary as this dynamic recalculation is done to avoid moving further into dead ends, and to avoid collisions.

<center>![](assets/images/finalchallenge/path_planner_tree.png =600x300)</center>
<center>_Figure 1: RRT Path Planning Tree in Simulation_</center>


#### Trajectory Following - Josh

Carefully navigating the narrow, unknown hallways of a labyrinth requires a very different version of pure pursuit than racing in a wide hallway.  Most importantly, our pure pursuit in the labyrinth has to be able to make three point turns.  Unlike in the race, there are cases in the labyrinth where the car may be facing the wrong way on a path and will not have enough room to make a full circle.  To implement three point turns, we initially developed a relatively simple algorithm that relied on the occupancy grid of the map.  In this algorithm, whenever the car detects that it will encounter an obstacle on the path to reach a point on the trajectory, the car will plot a course of successive arcs that alternate going forwards and backwards until the car determines that it can make an unobstructed turn and return to the trajectory.  While this approach worked reliably on static maps in simulation that the car knew entirely, using Google Cartographer to explore an unknown map introduced some difficulties.  Depending on what threshold value we set to determine if a pixel on the map was an obstacle, the car either crashed into walls that it did not think were obstacles or refused to plot a course through open space that it thought were obstacles.  When both these phenomena occurred repeatedly with a single threshold value, we realized we had to rethink our approach.  We redesigned the three point turning mechanism to only rely on local laser scan data.  To implement three point turning, we split our laser scan data into 4 regions, the region back-right of the car, front-right of the car, front-left of the car, and back-left of the car.  

<center>![](assets/images/finalchallenge/car_laser_scan_regions.png)</center>  
<center>_Figure 2: Laser Scan Regions_</center>

If the car was trying to reach a trajectory point to the left, it would turn the wheels towards the left while going forward and to the right while going backwards.  In this situation, we cared about the front-left laser and the back-right laser scan data as these would be the regions the car would enter while driving forwards or backwards.   If the car was twice as close to the front obstacle as the back obstacle, it would reverse, and if it was twice as close to the back obstacle as the front obstacle it would drive forward.  This allowed the car to successfully three point turn even in narrow areas without the uncertainty introduced by Google Cartographer.

### Safety Controller - Josh

One of the main underlying principles in our plan for getting out of the maze was to continuously explore without crashing.  To do this, we needed a safety controller to avoid obstacles.  Since we cared less about exploring intelligently and more about preventing crashes, our safety controller took a fairly simple form.  Whenever the car detected a nearby object in front of it, it would reverse in the opposite direction from where it came until it was a certain distance away from the object in front of it.  Turning the opposite direction prevented the car from getting stuck in a cycle of backing up and driving forward along the exact same route forever.  We set the threshold distance at which the car would stop backing up to be slightly larger than the distance from the LIDAR to the front of the car.  This prevented the car from backing up too far and hitting obstacles behind it at the cost of requiring more turns to get away from an obstacle.  We felt this tradeoff was acceptable in order to avoid collisions.


### ROS Implementation - Michelle

We utilized Google Cartographer in order to provide real-time 2D slam. The dynamic map along with the updated poses of the robot is used as the input into the path planning node. The path planning node dynamically calculates and recalculates an optimal route to get out of the maze. The pure pursuit follower allows the robot to follow this changing path. In addition, the pure pursuit controller is responsible for implementing a three point turn maneuver when it realizes that it can follow the path without running into obstacles. 


## Experimental Evaluation - Michelle

We evaluated the path planner and high-level maze solving strategy by testing the robot’s ability to navigate out of a wide variety of maps without excessive amounts of redundancy with re-exploring. We tested this is simulation using a variety of different maps. 

<center>![](assets/images/finalchallenge/final_map2.png)</center>
<center>_Figure 3: Simple Maze_</center>


<center>![](assets/images/finalchallenge/final_map4.png)</center>
<center>_Figure 4: Difficult Maze_</center>



### Testing Procedure - Victor

The testing procedure was done in simulation for each individual piece of our algorithm. The path planner, trajectory follower, and cartographer were all tested separately in simulation, and once those pieces each worked they were tested in combination in simulation. Only after the entire maze could be solved in simulation was it tested in the real world.

Once each piece was tested individually, trajectory following and path planning were tested together in simulation using Google Cartographer. This ensured that pure pursuit could use the ever changing path created by path planner to escape a maze on an unknown map. Once this was accomplished, it was tested and tuned in the real world.

#### Testing cartographer - Michelle

Since our maze solving strategy relies heavily on having an accurate map of the explored regions, it was important to evaluate cartographer’s performance on the car in order to inform our strategy. We tested which kinds of robot motions would lead to better cartographer maps, which gave us information about an optimal trajectory follower. We also tested what different obstacles would appear like on the map, such as chair legs and tables. 

#### Testing path planner - Victor
Path planner was tested in multiple steps. First the path planner was tested on a static map. This made sure that the algorithm accurately avoided obstacles, and was able to find a path through a maze in a static map that was known beforehand. Once this worked, the path planner was tested on a dynamic map slowly being explored by a bag file of the robot driving around the basement and google cartographer. This tested path planner’s ability to create a new path as it discovered an obstacle impeded the current path. Once this worked, the path planner was complete and ready to integrate with the other pieces of the project.

#### Testing pure pursuit - Josh 
To test the pure pursuit controller’s ability to follow trajectories in cramped conditions, we began testing on known static mazes we created.  We tuned the lookahead distance to allow the car to follow paths in front of it without cutting corners too aggressively and crashing into walls.  Similarly, we tested the three point turning to ensure the car could successfully exit a variety of dead ends.

<center><iframe src="https://drive.google.com/file/d/1Bz7fdPyGdnR7BnbLhB1lnvGW4qWjux6w/preview" width="640" height="480"></iframe></center>
<center>_Figure 5: Three Point Turning in Simulation_</center>

Once we were satisfied that pure pursuit worked on static maps, we introduced the uncertainty of Google Cartographer and ran the same tests.  During this phase of testing we uncovered that the map’s uncertainty about whether points were obstacles caused our three point turning to act unpredictably and unreliably.  Once we changed our three point turning algorithm, we again tested on static maps, then maps generated by Google Cartographer until the car consistently followed paths closely and executed three point turns without crashing.  At this point, we began testing three point turning on the car.  We built contrived paths for the car that caused the car to three point turn in a variety of situations.

### Results - Michelle, Josh

Overall results: 
Our overall maze solving algorithm was tested twice on the final challenge demo day, but we also tested the robot’s ability to navigate out of a wide variety of environments around MIT's campus. The robot was able to navigate out of a classroom full of irregularly shaped objects and moving people. In addition, the robot was able to successfully navigate out of a 25m radius in an arbitrary location on the ground floor of Stata. All of these tests were completed without the robot hitting any obstacles. This shows the algorithm's robustness to solving any unknown maze.  

<center><iframe src="https://drive.google.com/file/d/1sbqsnp_vu2I0s51DzV5Vtapwpb7ht2u5/preview" width="640" height="480"></iframe></center>
<center>_Figure 6: Robot Exiting Classroom_</center>

<center><iframe src="https://drive.google.com/file/d/1L39L84YL04375DohwTM6AiBhuz9Jg3T_/preview" width="640" height="480"></iframe></center>
<center>_Figure 7: Robot Exploring Stata Center_</center>


We ran the car in the simple maze shown in Figure 3 and calculated the time it took the car to escape the maze and the number of paths planned by the car.

<center>![](assets/images/finalchallenge/PathsPlanned.png)</center>
<center>_Figure 8: Number of Paths Planned to Escape Simple Maze in Simulation_</center>


<center>![](assets/images/finalchallenge/EscapeTime.png)</center>
<center>_Figure 9: Time to Escape Simple Maze in Simulation_</center>


As is clear from the graphs above, the car consistently escapes from the maze in a reasonable amount of time.  It replans the path many times as more of the maze is explored and paths are invalidated.  However, the speed of our path planner allows the car to continuously explore without having to sit and wait to receive a path.

We also evaluated the performance of cartographer, path planner, and pure pursuit individually. 

_Cartographer_

We made two essential observations while testing cartographer on the car. First of all, we noticed that slower motions where the robot completed loops resulted in a map free of loop closure errors when compared to fast linear motions. As a result, our trajectory follower uses a three point turn maneuver in order to complete as many loops as possible, resulting in an optimal map. Our second observation was that thin obstacles such as chair legs would show up really small on the cartographer map. We decided that we would have to dilate the map in order to avoid hitting the chair legs. 

_Path Planner_

It was essential for our path planner to be fast, so that we could spend a minimal amount of time waiting for paths. We measured the time it took to replan a path 14 times and we found that on average, we were able to replan a path in .241 seconds with a standard deviation of .255 seconds. Considering that our robot is going at a speed of .5m/s, this replanning time is extremely fast for our purposes. 

<center>![](assets/images/finalchallenge/TimeToPlanPath.png =752x452)</center>
<center>_Figure 10: Time to Replan Path_</center>

_Trajectory Following_

The primary goal for our trajectory following was that it’s able to navigate to its commanded path without hitting obstacles in all scenarios where it is physically possible. A secondary goal was making it take an efficient route to the path. Our three-tiered approach to trajectory follower was able to get the robot to follow any path without hitting obstacles most of the times. The only edge case where it could possibly hit a wall is if the safety controller reactively backs into a wall, which rarely happens. 

<center><iframe src="https://drive.google.com/file/d/1CqFnmCxltXjSYkoH8K2A4_Mqt8ANeINi/preview" width="640" height="480"></iframe></center>
<center>_Figure 11: Robot Successfully Exiting Maze on Final Challenge Day_</center>


## Lessons Learned - All

### Technical Conclusions - Josh

The main principle that trying to escape from the maze made clear is that technical implementations must be robust to real-world uncertainty.  Ideas that worked well in simulation were incredibly error-prone when tested in the real world. We learned that we may need to alter our initial strategy when faced with longer, more complex challenges, especially those that involve an uncertain environment. 

### CI Conclusions

1. Fredric Moezinia: It has taken us a few weeks but we are all very comfortable with each other now. This makes it easier to handle difficult situations, assert ourselves and challenge others. 

2. Victor Fink:  By this final project, we were very good at communicating effectively. We sufficiently divided up the work and accomplished goals on a timely basis. We could have done a bit better at getting started earlier on the project.

3. Josh Rosenkranz: In this lab, we did a far better job of working in parallel and communicating extensively about what the expected inputs and outputs of our modules were.  This made piecing together all of our contributions so much easier and reduced the amount of intra-group conflict we experienced near the end of the project.

4. Charlotte Sun: It’s both exciting and sad knowing this is our last lab for the semester. We all spent a lot of time and worked really hard to complete this final maze challenge. We had two teams working in parallel on different tasks, but we communicated well and kept everyone on the team updated. I do wish that we could've decided on a path planning algorithm earlier so that we could integrate all components together and start testing earlier.  

5. Michelle Tan: We were able to successfully delegate different portions of the challenge to different people, and the parts were successfully integrated just before the day of the final challenge. However, our timeline for this lab was pretty suboptimal considering we were forced to debug the entire integrated pipeline in the two days before the final challenge, which was pretty stressful. 


## Future Work - Josh

Though our car successfully navigated out of the maze, our overall implementation still has plenty of room for improvement.

_Path Planner_

To speed up our path planner, we initially tried to store a tree of points generated by RRT and then prune branches that were discovered to be invalid.  Maintaining this logic ultimately proved too difficult to debug so we rebuilt the tree from scratch for every path.  Successfully storing this tree would improve the speed of the path planner.  To improve the ability of the path planner to plan paths out of the maze, we could implement the bridge test.  This would result in a higher density of points near narrow openings, such as the exit of the maze.  We could also bias the path planner to look for paths through unexplored regions of the maze to minimize the amount of time we spend repeatedly exploring areas.

_Trajectory Following_

Three point turning and the safety controller could also be made more intelligent.  Three point turning could do more processing of the laser scan data to detect obstacles and more smoothly execute a turn that minimizes the back and forth movement.   The safety controller could incorporate previous occupancy grids into its decision making to ensure it does not back into obstacles.

