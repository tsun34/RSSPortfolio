Lab 5
=====

## Overview and Motivations - Charlotte

Localization and mapping are the last two components of perception in the robot architecture. In lab 5, we solved the localization problem by implementing Monte Carlo Localization (MCL), also known as the particle filter. We were able to determine the robot’s position and orientation in a given map using the distance readings from LIDAR. Combining localization and the computer vision model we implemented in lab 4, we now have a completed perception model that processes signals from sensors and estimates the state of the robot. In the rest of the semester, we will switch gears and focus on the development of planning and high-level control.

## Proposed Approach - Charlotte

Monte Carlo Localization (MCL), or the particle filter algorithm, recursively conditions belief on directly observable system variables, such as the distance data from the LIDAR, in order to infer unobservable state variables, such as the robot’s exact position and orientation on a given map. The mathematical principle behind the MCL algorithm are the Markov Assumption and Bayes’ Rule. It represents the belief (aka. the robot position and orientation) distribution as a set of weighted samples. It is designed such that better estimations of particle positions will have a higher likelihood of being sampled in the next iteration. 


### Initial Setup - Victor

The localization in MCL is based off of two main parts: a sensor model, and a motion model. The sensor model updates the likelihood that the robot is at each pose given the robot’s sensor readings and the expected sensor readings at that pose.  The motion model takes in the current pose of each particle and a change in pose to create a new projected pose for each particle. These models work together to accurately localize the robot.


### Technical Approach 

#### Sensor Model - Josh

<center>![](assets/images/lab5/sensor_model_2D_4-1-18.png =500x500)</center>
<center>_Figure 1 : 2D Slice of Precomputed Sensor Model_</center> 

<center>![](assets/images/lab5/sensor_model_3D_4-1-18.png =500x500)</center>
<center>_Figure 2 : 3D Precomputed Sensor Model_</center> 

To create a model of how likely different laser scan readings were at varying distances, we split up possible laser scan readings into four categories:  known obstacles, unknown obstacles, missed readings, and random measurements.  The known obstacles category represented laser scans that detected known objects on the map.  These readings are expressed as a Gaussian centered at the expected distance to the object with a small standard deviation, sigma. Unknown obstacles are laser scans that detect objects not shown on the map but that may be near the car, such as a person walking in front of the car.  These readings are parameterized by a y-intercept, b, representing the likelihood of a reading very close to the sensor, and a negative slope, m, representing the decreasing likelihood of scans at further distances.  The third category, missed readings, was for laser scans that returned the maximum distance because of an error in detecting the returning laser.  This was simply a probability value, max\_prob,  added at the max laser scan range value of 10 meters.  Lastly, the random measurements captured any random readings the laser scanner may return.  This was represented as a uniform low probability, random\_prob, across all possible reading distances.  Thus, the sensor model was parameterized by five values, sigma, b, m, max\_prob, and random\_prob in the following manner:

Probability(getting reading x given car is at distance d) = probability of detecting known obstacle at distance x + probability of detecting unknown obstacle at distance x + probability of missed reading (if x = 10) + probability of random measurement.  In mathematical form, 

	Pr(x | d) = 1/(2*pi*sigma^2)*e^((-(x-d)^2)/(2*sigma^2)) + max(0, m*x + b) + max_prob (if x = 10,  else 0) + random_prob 

Once all the localization code was functioning, we were able to optimize these five parameters to best minimize the localization error of the car.  After generating the probabilities, we raised the entire distribution to the ⅓ power.  This had two benefits.  First, it increased our minimum probability to on the order of 0.1, rather than 0.001.  This reduced the likelihood of running into float precision errors caused by multiplying very small floats together.  Also, raising the distribution to the ⅓ power made the distribution less peaked, allowing for a more robust set of probabilities.

#### Motion Model - Victor
The second piece of the localization is the motion model, which takes in our assumed current pose and an approximate change in pose and outputs a new pose. First, our model takes in the twist data from the Odometry message that is published to vesc/odom from our teleop node. The twist represents the linear and angular velocities according to our robot’s sensor. The linear velocities include velocities in the x (forward or backwards), y (left or right), and z (up or down) directions. These we refer to as x\_lin\_vel, y\_lin\_vel, and z\_lin\_vel. The angular velocities were in the x (twist along the y-z plane), y (twist along the x-z plane), and z (twist along the x-y plane) directions. These we refer to as x\_ang\_vel, y\_ang\_vel, and z\_ang\_vel. Since there the car never moves up or down, the values for z\_lin\_vel, x\_ang\_vel, and y\_ang\_vel are always 0. The only change in theta, or the angle of our pose, is the z\_ang\_vel times the time between Odometry messages for every particle, or 

	time_between_messages * z_ang_vel

The x and y positions of our racecar represent the x and y position on our map, while the x and y linear velocities represent x and y changes relative to the robots current direction. Therefore for each particle, the change in the x direction is 

	time_between_messages * (x_lin_vel * cos(theta) + y_lin_vel * sin(theta))  

and the change in the y direction is 

	time_between_messages * (x_lin_vel * sin(theta) + y_lin_vel * cos(theta))  

Since the car is always moving forward, and there are so many Odometry messages per second, the y\_lin\_vel term is incredibly small compared to the x\_lin\_vel even though the car turns. Therefore the y\_lin\_vel term is almost negligible in our model and has almost no impact on the accuracy of the motion model.

We attempted to use the pose of our odometry messages instead of the twist, and simply calculate change in pose between time steps. This resulted in worse localization. With everything else controlled, our car ran at the same rate with both the twist method and the pose method applied, yet on the autograder the pose method for our motion model regularly scored less than 75% of the score we got with the twist method applied. At the time of this testing, our localization model was running at a speed of 26 Hz, and the autograder applied to the twist method averaged a score of .48/1.0 while the autograder applied to the pose method averaged a score of .34/1.0. Perhaps this was due to the implementation of the pose method, but either way this led us to continue using and refining the motion model based on the twist of the Odometry message.


### ROS Implementation - Michelle

An essential part of this lab was structuring our program to be both performant and easy to debug. In order to optimize for speed, we minimized function call overhead on the critical path. We also made sure to initialize all memory once at the beginning. For debugging purposes, we added visualization outputs of the predicted particles and inferred pose into rviz to easily verify the accuracy of our code. 


## Experimental Evaluation - Michelle

In order to evaluate the correctness of our localization, we ran our algorithm in both the simulator and on the robot. 

### Testing Procedure - Victor

For our localization function to work correctly we needed it to both run at a rate close enough to 40Hz and to localize the particles correctly within the map. Therefore the testing and optimizing of the function was twofold: trying to make it run fast enough and trying to make it run accurately enough. Sometimes, one would affect the other.

The first objective was to get the particles to appear in the rviz map, and to respond to both Odometry messages and LaserScan messages that the Particle Filter subscribes to. For this, we ran our localization function and used clicked pose to initialize a pose in the map. Once this manual method worked, we tested to make sure that the autograder’s initial pose messages would also be properly used. To do this we tested using the autograder so that the particles would be initialized without a manual click.

Once the autograder was able to run, this told us that our code was set up to be optimized. It gave us an idea of how accurately our code returned the pose of the robot, and it told us exactly how fast our function was running. All of the testing and optimization for speed was based off of the rate recorded by the autograder. We used this rate and a flame graph to determine the bottlenecks of our program and how to adjust accordingly until our program ran as close as possible to real time.

<center>![](assets/images/lab5/Flames.png =700x500)</center>
<center>_Figure 3 : Flame Graph of Particle Filter_</center> 

We used the score of the autograder, as well as the Rviz visualization to debug and optimize the accuracy of our program. Small tweaks to our parameters increased our score immensely once our program somewhat localized correctly.


### Results - Michelle

#### Simulator: 
The robot pose estimated by our localization fits well with the ground truth provided by the simulator. In the simulator, the simulator pose represents the ground truth location of the robot. When visualizing both the simulator pose and the estimated pose from our localization, the poses match up well.

<center><iframe width="560" height="315" src="https://www.youtube.com/embed/t8xwYJI4tZI" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe></center>
<center>_Figure 4 : Localization Performance in Simulation_</center> 

<center>![](assets/images/lab5/graph.jpg =500x400)</center>
<center>_Figure 5 : Difference Between Ground Truth and Estimated Pose_</center> 

This graph shows that the estimated position of the robot matches closely with the robot as it drives in the simulation. After initializing a pose, it converges quickly to the correct position. The actual errors were on the order of centimeters. 

#### Robot in real world:
The pose in the stata basement estimated by our localization matches well with the actual location of the robot in the stata basement. In the video below, the location of the robot in the real world visually matches up with the particles in rviz. 

<center><iframe width="560" height="315" src="https://www.youtube.com/embed/kQU7Iwb_vPE" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe></center>
<center>_Figure 6 : Localization in the Real World_</center>

Although it’s hard to measure the exact location of the car precisely, using visual inspection, the video verifies that our pose estimate is reasonable. It moves at a similar trajectory to the robot and doesn’t drift as time goes on.  Additionally, landmarks that the car passes in the basement match up with landmarks passed in the simulator. The laser scan from the robot matches up well with the walls of the map. 


## Lessons Learned - Fredric

As usual, we found that physically working together was best. This is a lesson which has been reinforced to us lab after lab. We always seem to be more productive as a team when we work together.

Secondly, we acted on our previous mistakes and did a much better job at getting our priorities in order earlier. Since we were able to plan earlier and more rigorously for this longer lab, we learned that this strategy helps a lot during the later stages. Using tools like Trello and Slack helped us make much more progress much faster.

We learned that producing a semi-pseudo version of the lab’s software portion is not particularly helpful. This means that having one person attempting to code the entire code pipeline might actually cause more problems than it solves. We learned to modularize and stick to a delegation of tasks.


### Technical Conclusions - Fredric

This lab included much more TA code compared to previous labs. This is both helpful but also misleading in certain cases. We choses to prioritize our own thought out strategy rather than try and conform our code to the TAs. This worked well and we concluded that we will use this strategy in the future.

We also learned that the TA’s code might not always be correct. When a ROS correction of queue size was made by a TA, we did not pay enough attention to the reason behind this correction. This wasted many hours trying to find a different bug that did not exist. In conclusion, we learned to check and validate all TA code given to us.

Finally, we learned the importance of tuning and optimization. Optimizing is a more deterministic process which we did quite well. We learned, however, that tuning is not just an afterthought. We have to build code in a way that will let us tune parameters and other code snippets easily in the future. This will help a lot with analysing the effect of tuning certain parameters and variables.



### CI Conclusions - All

1. Fredric: This lab was longer and harder than the others. It required more planning, and using Trello helped us. Personally, I didn’t contribute as much to this lab as past labs; I made up for a busy week by trying to roughly code up the entire lab by myself at the beginning. I have learned that this is unhelpful, and I shan’t do it again!

2. Victor: This lab was tough because there was a lot we did not understand at first, and once we did understand it we had a ton of errors. There was a lot of confusion and wasted time because of a lack of effective communication in what had already been implemented and what had not been. Pseudo code tripped us up a bit. We need to communicate more effectively in the future.


3. Josh: We effectively split up tasks initially for this lab, but we struggling to integrate the different pieces into one cohesive code base.  Everyone worked well on their tasks but we did not communicate well enough about the expected inputs and outputs for each module.  We wasted a fair amount of time because we were not sure how different pieces of code were supposed to fit together and whether chunks of code were complete or not.


4. Charlotte: Lab 5 is the first lab I did after joining team 13. It has been a great pleasure because every team member is very welcoming and supportive, especially when lab 5 is much longer and more difficult compare to previous labs. Unfortunately, as a new member, I’m still trying to become familiar with the team’s workflow and catching up my pace. I’ll try to be more initiative and take on more active roles in future labs.

5. Michelle: This lab was complicated and had a lot of different parts to it, which made it even more essential to communicate well within our team. There were times these two weeks that were pretty rough, but in the end, our final product turned out well. 

