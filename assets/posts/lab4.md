Lab 4
=====

## Overview and Motivations - Fredric

This lab was our first opportunity to incorporate objection detection into our robot software. Getting this right allows us to control the motion and path of our racecar based on the robot’s environment. The lab consisted of four distinct, yet intertwined parts. The overarching goal is for the robot to autonomously locate and park in front of a cone, and follow a tape line on the ground.

The first was testing and implementing cone detection algorithms to be able to use the Robot’s Zed Camera to ‘see’ a bright orange cone. Once this is detected, we want to extrapolate the location of the cone relative to the robot. We can then use this knowledge to drive up to and park the robot. We can also use a variant of this to have our racecar follow a circular tape line at different velocities.

The nature of this lab lends itself to a modularized codebase, yet one that relies on well-connected communication since information is vital as it travels down the pipeline. Overall, we managed to parallelize the software construction in an effective and comprehensive manner.
This will make it easier when incorporating this lab into the final racecar race. We imagine that one of the most vital components of making a successful racecar will be figuring out a driving path based on certain objects to follow and avoid.

## Proposed Approach - Victor

Our approach consisted of three main steps; detecting the location of the object in the picture, converting the picture coordinates into real world coordinates, and moving the car based off of the coordinates of the object. Both line following and cone parking used these same three steps, but had different implementations to accurately achieve both functionalities.

### Initial Setup - Victor

We built four nodes for implementing this lab. Two were for object detection, one for detecting a cone and one for detecting a line. The other two were for moving our robot, one each for parking in front of the cone and following the line.

### Technical Approach - Michelle, Josh, Fredric, Victor

In order to detect the orange cone, we used OpenCV to process the data from the ZED Camera.  We evaluated three common object detection algorithms: SIFT/RANSAC, template matching and colour segmentation. After evaluating IoU metrics for the three algorithms, we determined that the  color segmentation algorithm outperformed the others.

SIFT/RANSAC:
SIFT/RANSAC, although being a popular object detection method, worked very poorly in this situation. The algorithm depends on being able to match distinct features in two images, such as prominent edges and corners. However, since the cone is single-colored, there aren’t very many keypoints to match. This image shows the keypoints found in both the template and image where the size of the circles are the strength of the keypoint and the line in the circle represents the direction of the keypoint. As shown, the cone doesn't have very many features to detect, so SIFT works poorly.

*Visualization of Keypoints on Cone using SIFT/RANSAC*

![](assets/images/lab4/SIFT.png =700x300)

Template matching:    
Template matching was both too slow and inconsistent to be feasible to detect the cone. Template matching uses a template and looks for a group of pixels in an image that are similar to the template image. This algorithm ran the slowest at approximately 0.25 seconds, since the code had to iterate over search for different sizes of cones. This would have caused a lot of lag in a real-time situation. In addition, the algorithm is extremely error prone to varying orientations of the cone. However, we want our robot to detect the cone no matter how it’s rotated.

Color Segmentation:      
Color Segmentation works well in this situation because the cone is bright orange, which generally contrasts colors in the background. In addition, since the cone is all one color, it’s easy to select for the cone just by filtering for a range of values. However, since the algorithm is just looking for colors, if the background contains colors similar to the cone, the robot may not be able to distinguish between them.

*Dynamic Detection of Cone using Color Segmentation*

![](https://media.giphy.com/media/4QFdhC9YFtMEsiwvyu/giphy.gif)


Object Location:     
To determine the location of the object, we utilized a homography matrix that mapped pixel locations to real world (x, y) coordinates.  We took sixteen sample measurements of the cone in different real world (x, y) locations and determined the pixel locations of the base of the cone.  We then used a built-in OpenCV function to find the homography matrix that mapped the pixel locations to real locations.  The function found the homography matrix that minimized the square error of the resulting projections.  From there, we simply took the bottom of the bounding box as the y-coordinate and the average of the left and right sides of the bounding box as the x-coordinate of the base of the cone in pixels.  We used our homography matrix to convert these pixel locations into (x, y) locations, which were then published to the object location topic.

*Checkerboard used for computing homography matrix with*
*(u,v) pixel locations and (x,y) real world locations of each cone placement*

![](assets/images/lab4/homography_graph.png =500x190)

For our first goal of parking the robot, we used the x,y coordinates published by the object location module. We implemented a simple proportional controller for speed and steering angle similar to the Wall Follower Controller in order to guide the robot within two feet of the cone.  To determine velocity, the P controller simply found the distance from the cone, subtracted the desired distance, and multiplied this error by a velocity gain.  For finding the steering angle, the heading to the cone was just multiplied by a steering gain.  Testing in simulation and on the real robot allowed us to fine-tune these gain values to achieve quick convergence with minimal oscillation.

*Successful Car Line Following*

![](https://media.giphy.com/media/dJHpuM56vksagi3OoA/giphy.gif)

Finally, we implemented line following using a similar strategy. We were able to use much of the same code to do this task, although very different in nature. We blacked out parts of the incoming image, and used color segmentation to find the location of the line to follow. For line following, the robot must constantly move forward, so we approximated a y value (x being directly ahead of the robot) which we proportionally based our steering angle and velocity off of. Instead of using a homography matrix, the y value was approximated based off of the midpoint of the u values of our bounding box and scaled into a value to directly represent the necessary steering angle. The larger the steering angle, the smaller the velocity, so that the car could easily traverse curves in the line.


### ROS Implementation - Josh

We attempted to generate modules to better reason about the implementation and to allow for parallelization of tasks. We split our ROS implementation into several packages, one for detecting the object, one for parking, one for detecting the line, and one for following the line.  Within each of these packages, we also separated the code into modular functions.  For instance, within the detect object package, there were separate functions for finding the bounding box and converting the bounding box into a real world position of the cone.

*Flow of Information through ROS Topics*

![](assets/images/lab4/information_flow.png =350x95)

## Experimental Evaluation - Victor

This lab was more difficult to test and tune than past labs. Depending on the environment we were running in and the lighting, we had to adjust our thresholds for the color values. We managed to get our car working correctly, but had a lot more hurdles and bugs than last week.

### Testing Procedure - Victor and Michelle

During this lab, we had several issues with our robot. First, we could not connect to the robot via the router, and then we had the Zed camera preventing joystick control. These challenges made it incredibly difficult to test on the robot, so this emphasized the practice of testing in simulation first. For testing the cone detection, we collected bag files from the robot in order to visualize a bounding box around the cone. For testing the homography matrix, we used rviz to visualize the real-world location of the code. In order to test the parking, we used an interactive marker in rviz that we could drag around to represent the cone.

*Simulation of Car Parking*

![](https://media.giphy.com/media/8lKyjdROuOhPEusEk9/giphy.gif)

Testing the object detection was relatively easy. We took in pictures from the robot's camera, and showed our bounding box over the image. This easily showed us where the robot thought the object was, and how we could adjust it to make it more robust.

Once the simulations ran well for the cone parking, and our hardware issues were fixed, we ran tests in the real world environment. The gain values needed to be tweaked when going from simulation to real world. At first the car’s motion was a jerky, but after changing parameters and a few minutes of testing this problem was not too challenging to solve.

### Results - Victor

The car parking started out jerky, but after some minor adjustments to the speed and steering angle gains the car runs much more smoothly. There are still a few issues with the jerkiness and if we need a more robust parking package in the future, we will take a short term running average of the perceived location of the cone in order to minimize this jerkiness.

Once the car parked well, we recycled a lot of the code into the line follower. Getting the line follower to work was a lot simpler than the cone parking because the car is constantly moving. Almost all of the required computation was focused on the steering angle. After we adjusted our gains once again we were able to get the car to follow both the simple circle line in lab, and some more complex lines.


## Lessons Learned - Victor

One thing we learned, that we can't technically control is the importance of a working robot. We had issues with our hardware on Friday and Saturday and it realy set us back in terms of our progress. It is important to make sure the robot works correctly early on in the week, and to maintain that functionality so that we can spend our time on the lab.

The second lesson we learned is the importance of working together. Some pieces of the lab were originally implemented by a single member and it made it difficult for another member to edit their code when they did not know how it worked in the first place. We need to work together and communicate better to use our time more effectively.

Lastly, we learned we have to get our priorities straight. We speng a lot of time trying to optimize cone parking before we even started line following. It is best to get a working product, and then once we have finished the necessary pieces, if we have extra time we can then go back and optimize our code.

### Technical Conclusions - Michelle and Josh
We ran into many hardware malfunctions have make it necessary to find other ways to test our code, including collecting bag files, using debugging outputs such as rviz, and using the simulator.  One major design decision that influenced our work on this lab was trying to find the simplest solution.  We easily could have over-complicated our design and tried to, for example, implement a full PID controller for parking.  Instead, we decided to implement a simple design, test its effectiveness, and only then add complexity to the design if we needed to.  This guided our design of the P controller for parking, the use of color segmentation for cone detection, and our determination of where the line was for line following.  Line following was one instance where our simplest design did not work at first.  We initially set speed to a constant value, but found that the car could not follow tight curves in the line.  After discussion, we decided to set speed relative to the how sharp the turn was.  This made the car slow down around tight turns and allowed it to track the line closely.


### CI Conclusions - [All]

1. Fredric: Due to our busy schedules, we rarely had full attendance at our meetings this week. We did show, however, that even 2 or 3 people working together can go a long way. We did a good job at updating the members who were not present, but we can always take this further and make sure everyone is on the same page!
2. Victor: We were not able to be fully together for most of the week, and a lot of times we had to communicate over slack. This made things a bit more difficult, especially when one member of the team implemented one piece of the lab and then was not around to explain it to other members. In the future, it would be best to try to do things together, or try to explain how something works better before we split up.
3. Josh: However, we did do a good job implementing modularity in our code for this lab.  We set clear expectations of what each module would get as input and what it should produce as output.  This allowed us to work independently on different sections of the code and then piece our modules together to form a complete pipeline from input image to driving command.
4. Michelle: The way we parallelized our work was essential to our success in this lab. Despite only having several hours this week when all of us could physically be in the same location, we were able to make progress. In the end, everyone implemented the sections of the lab that they were responsible for. However, the most time was spent on communicating how we would fit these separate sections together.
