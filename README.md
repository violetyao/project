# Lychee Bot - Group 36 EE106A Final Project

lychee-bot.xyz

## Table of Contents
1. Overview and motivation
2. Research Goals
3. Design
4. Data Collection
5. Analysis of human motion
6. Human Model
7. Trajectory Planning
8. Actuation
9. Challenges
10. Future Work
11. Team
12. Resources

## Overview and motivation:

Autonomous navigation in an environment with humans is challenging but can be highly useful for applications such as food delivery (like our inspiration Kiwi bot) and navigating in homes or offices. Our research project focused on modelling human-human interaction and robotic autonomous navigation among humans.

Navigating among humans is challenging primarily because of the difficulty in predicting human motion and their interactions with others. For an external agent such as a robot that does not know the goal state of the human or other prior context, it is even more challenging to predict human motion, and navigate around them to reach the goal. We aimed to experiment with various approaches and create a prototype to achieve this goal. 

This research project has three main parts: human modeling, path planning and robot control. For modelling the human-human interaction, we analyze the takes we recorded in the Cory motion capture room to infer patterns, and apply the Maximum Likelihood Constraint Inference for Inverse Reinforcement Learning paper to determine constraints in human interaction. We also developed and experimented with an MCMC based approach to predict human motion. 

For the planning component, we created a constraint optimization based planner. Finally, we developed a controller for the Turtlebot to navigate around obstacles detected by the OptiTrack motion capture system. The controller is modular and can take in commands from any path planner. 

## Research Goals:

To summarize our we have 4 stages that we would like to achieve:

1. Understand how humans interact with static obstacles as well as other humans.
2. Path plan a simple trajectory from current state state to goal navigating around static and moving obstacles. 
3. Model a human’s trajectory and incorporate its behavior into our path planning algorithm.
4. Create a controller to receive commands from a master algorithm and actuate.

## Design:

As it is a research project, an important goal was to experiment with different approaches while making sure to have useful results. At a high level, our project incorporates sensing, planning, actuation and learning. We aimed to modularize our design so we can work on it parallely, while some aspects such as the Turtlebot, was not available in the motion capture room. 

Since the goal is to make a robot navigate to its goal around humans and our research goal was to experiment with methods to model humans, we made design choices to focus more on these novel problems. Choosing the OptiTrack motion capture system to locate agents and obstacles, instead of an on-board camera, allowed us to have a comprehensive localization and mapping system, while allowing us to focus on the research problems.     

We decided to use the OptiTrack motion capture system rather than use computer vision to detect humans and other obstacles to get more accurate object positions and  a comprehensive map. Initially, we had used the Turtlebots’ Kinect sensor to detect and avoid obstacles but we quickly realized that it would an ineffective way to address the primary research questions (since it needs to move around to generate the map, account for errors, detect occluded humans). 

## Data Collection
We wanted our data collection to be holistic so that it would provide us with a good overview of how humans navigated around obstacles. Thats why we did 6 different collisions with different subjects and multiple trials. 
[Description of different paths](https://github.com/Lychee-Bot/project/blob/master/Project%20Idea.pdf)

Collecting this data was essential in the human modelling and to accomplish this we used the optitrack system. There were several challenges with data collection though the primary ones being:
1. We had to learn how to create rigid bodies (of the caps that were used to track the particpants) and add them to the optitrack system.
2. Formatting and null values of the data - the data provided was extremely poorly formatted and to automate graph generations we had to build a pipeline to clean (remove null values or fill wherever possible!)

### add videos and pictures^^^

## Analysis of human motion

Based on primliminary analysis with 20 video takes in the Motion Capture room of 2 people moving. Human motion is dynamic and highly context dependent, however, we wanted tried to infer simple patterns in the motion.
Here are a few observations from the data:
- Only one person tends to move away from the other when there is a head-on collision. 
- When moving at higher speeds the movement away from the person is closer to the time of collision but the deviation is much higher. 
- People prefer maintaining a longer backward distance than a sideway distance when walking next to people.

![Image](https://github.com/Lychee-Bot/project/blob/master/img1.png)	
- This image shows how the horizontal distance between two people changed as the collison was about to occur. You can clearly see how the horizontal distance in mantained until the time of the collision where both the participants turn away from each other. 
- You can also notice how there was some hesitance on who was going to move out of whose way at around 1.3 seconds, but ultimately they deviate away from each other before the collision which occured at 1.8 seconds as seen in the graph below.

This analysis also helped us calculate useful metrics across the samples: the median minimum distance between people is: 0.469 m, standard deviation: 0.195. The median minimum distance is used in our current path planning algorithm as a hard constraint. 

![Image](https://github.com/Lychee-Bot/project/blob/master/img2.png)	

## Human Model

We read various papers on human motion modelling and we experimented with the approach that our mentor Dexter Scobee had described in the paper [Maximum Likelihood Constraint Inference for Inverse Reinforcement Learning (D.Scobee, S. Shastry)](https://arxiv.org/abs/1909.05477).  This paper reformulates the Inverse Reinforcement Learning problem to describe the observed behavior with a simple reward and a set of hard constraints. 
We used this paper and Dexter's repository to identify constraints for moving humans. 

Another approach that we read about was social spheres also known as proxemics. In this area of research the leading school of thought is that humans maintain a relatively fixed set of distances from each other when navigating through urban environments. Even though this set of distances is unique to each human, research has shown that these proxemics are usually ellipsoids. This is somewhat consistent with our analysis where we found that people tend to maintain a longer vertical distance than a horizaontal one from each other. Ultimately we didn't decide to go down this approach as we didn't have enough data samples, especially un-tainted ones (not from a lab setting) to create such a generalization and it would be extremely complex for the given time frame to do so. 

In the input example we used, when two humans start by facing each other, and move across to the other side, they are close to each other towards the middle of the room. The other human's position near the potential head on collision time is a constraint that we want this IRL constraint inference model to identify. 

There are two ways in which constraints inference is extremely useful to us -
1. It allows us to infer static obstacles such as puddles or broken glass on the floor that a robot may not be able to detect. By observing the human's movement, we could potentially detect these constraints while path planning.
2. Robots have limited range of vision and may not be able to detect obstacles that are far away. By looking at the movement of the humans and sudden changes in its path (for example when two humans are having a head on collison), can allow the robot to potentially infer other moving obstacles.

To get the IRL algorithm to work we had to - 
1. Create a MDP of the Cory lab with states, actions, and relevant time discounted rewards (we took Dexter's help in formalizing our MDP problem)
2. Discretization of the human modelling data so that we could fit it to the MDP and track the movement of the person through the grid. We used several techniques such as moving averages to get a much smoother path when transitiong from one state to another so that we could get a realistic path.This would also make our path generation more robust to optitrack errors!
3. To generate (state, action) pairs we had to write a script that given the layout of an MDP and a list of states, it would generate the corresponding actions.
4. After discretizing our data, and putting the it into relevant (state,action) pairs we were able to use the MDP to infer constraints.

Key Facts about the MDP and model:
1. num_states: 81
2. Transitions allowed: allow_diagonal_transitions = true; num_actions = 8; up = 1; down = 2; left = 3; right = 4; up_left = 5; down_left = 6; up_right = 7; down_right = 8
3. Goal and end state got from discretization of the data
4. Max path length: 15
5. Discount factor 0.9

## Trajectory Planning

To accommodate for potential changes in human motion prediction, trajectories are planned by solving an optimization problem as prediction updates.

![Image](https://github.com/Lychee-Bot/project/blob/master/Loop.jpg)

In our update loop, an initial trajectory-planning problem is solved with the result of a first attempt to predict human trajectory. The program then enters a feedback loop where whether goal is reached (within some acceptable margin) is checked. At the beginning of each time interval, an update for human trajectory prediction is retrieved and checked against the previous prediction. If the previous one is deemed as off compared to the updated one, the trajectory planned and currently executing will be deemed invalid. The trajectory-planning problem is solved again using the TurtleBot's current location as the starting point and the updated human trajectory prediction as part of the constraints. If the maximum time allowed to reach goal is exceeded, the program will instruct the TurtleBot to halt at its current location and return a failure flag, so that manual control can be used to navigate.

## Actuation

We have thought about two different approaches to achieve our goals. Approach 1 leverages existing turtlebot pacakge, but we soon found out a major drawback that prevent us using approach 1, so we switch to approach 2. In Approach 2, we set up our own master server that compute the path and send commands to our controller based on human modeling constraints.

### Approach 1: Map Update
The first time we thought about "turtelbot avoids pedestrain and goes to a specific location on map" problem, we were confident that there must be online solutions, since the turtlebot is very popular. Indeed, there is one path planning package called ```actionlib``` which takes a stacic map and a goal (inspired by [Learn Turtlebot and ROS](https://github.com/markwsilliman/turtlebot/)). It has a built-in function that will compute the path for turtlebot, and moves the turtlebot to desired location. However, this method doesn't do well in our goal: 

* The map is static. In our case, pedestrain is always moving. A static map doesn't work.
* The computed path may not be our desired path. From our human modeling, we find the optimal threshold of distance turtlebot has to keep from pedestrain. The path generated by ```actionlib``` may not obey this rule.

We then come up with a modifeid solution to use ```actionlib``` package: considering pedestrain as a moving obstacle (circle, with the radius as our distance threshold) [**insert picture here, caption: we write a python script to generate pgm map with obstacles**] Each timesteps we will update the map by moving the circle. The steps are:

1. Create a map of the mocap room, use ```actionlib``` package to generate a path.
2. Add pedestrain as a moving circular obstacle, and update (generate new map) the circle every time-step. Call ```actionlib``` again to update our path
3. Turtlebot moves based on current path command.
4. Repeat step 2 and step 3 untill turtlebot reaches goal location.

We test our approach first by running ```rosrun map_server map_saver``` to create a layour map (pgm image), and then modify the map with image editing tools to add walls (black region in the map). Second, we run the ```actionlib``` code that go from anywhere to a fixed location on both map. As we expected [**insert picture here**], in the left map the turtlebot could move freely, and in the right map turtlebot will stop (no path could be found because of the wall). Our modification of the map works! However, we soon realize a **critical drawback** of this approach: the ```actionlib``` server will send commands all at once! Even we could update the map, the ```actionlib``` server itself will not update the path! Searching online about solution extensively, we soon realize that our approach works best with Lidar for SLAM algorithm. Since we don't need to use lidars, we give up this approach.

### Approach 2: Master Action Server
Our second idea comes from looking into the ```actionlib``` package. This packages takes the map information and computes the path in an action server and then sends moving commands to the local turtlebots. We could adopt this idea too! Our problems therefore boils down to three subquestions: 
1. design our own localization
2. path planning
3. controller modules.
Luckily, we could use the optitrack system as a localization method, our research on  human modeling and prediction as a mathod for path planning, and use [Learn Turtlebot and ROS](https://github.com/markwsilliman/turtlebot/) moving function as a starting point for the controller. [**insert image here**]

## Optitrack System
The [Optitrack System](https://optitrack.com/) [pictures of the optitrack room] is an advanced motion capture system. It utilizes an infrared camera to localize the objects by tracking the special IR-reflective markers. It is a high-accuracy and low-latency system that fits our project's needs. 

Before we could begun the localization, we had to 
1. stick markers to the object we want to track on. 
2. group the markers together as "rigid body." 

Setting up the rigid bodies, allows the optitrack system to automatically broadcast the location of the objects to the network. We used the ```mocap_optitrack``` pacakge to receive the data.

### Localization
We leverage the Optitrack system to do the localization for TurtleBot and pedestrian. The TurtleBot will have markers on the top, and pedestrain will wear a cap with markers attach to it. Once the system finds TurtleBot and the cap (pedestrain), it automatically broadcasts the location data to the same network. Note that right now TurtleBot, instead of the Ros Computer,  receives the localization data so the previous ```mocap_optitrack``` package doesn't work on the TurtleBot. We searched online and found a nice ROS package for receiving such message on TurtleBot: ```vrpn_client```. Leveraging ```vrpn_client``` package we were able to receive the location message by simply subscribing to ```/vrpn_client_node/turtlebot/pose``` and ```/vrpn_client_node/cap/pose```. The returned message is a [PoseStamped](http://docs.ros.org/melodic/api/geometry_msgs/html/msg/PoseStamped.html) type of message. [**insert image here**]

## Challenges
1. Turtlebot access in mocap room only three days before!!! This hindered us from implementing the design that we had orginally forseen at the beginning of the project. Primarily not being able to connect the optimized path planning to the  bots.
2. Existing path planning package doesn’t work with Optitrack system
3. Optitrack and turtlebot Odometry system has two different coordinates
4. ROS_MASTER_URI conflict between turtlebot and optitrack
5. No way besides SLAM to update the map and do path planning efficiently

## Future Work
1. Research in smoothing the path (move in curves)
2. Perform online multi-human motion prediction and add to Turtlebot.
3. Add Computer Vision based planner. Test it on Berkeley Campus. 
4. Make Kiwi bot autonomous, then start our own self-driving delivery service :)

## Team
Mentors: Dextor Scobee, David McPherson

Name - Bio - what you did in the project - Favorite robot
1. Ganeshkumar Ashokavardhanan: Worked on human modelling, data analysis and the initial Turtlebot ROS controller. EECS & Business '21. Favorite robot: [Spot](https://www.bostondynamics.com/spot)
2. Shivam Shorewala: Worked on the human modelling side of things, ranging from data collection, to creating MDPs to speed analysis. EECS and Business 2021. Favourite robot [Mr.Robot](https://www.usanetwork.com/mrrobot)
3.
4.
5.
