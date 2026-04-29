# micromouseproject FDU

Micromouse Project Repository for FDU
# READ ME

## Intro 
Hello and welcome!
This is an info page for the Micromouse project. The purpose of the project is to build a robot capable of solving a maze as fast as possible.
The skills needed for this project are used in Robotics, System Engineering, Computer Programming, and Hardware Design among others.

## A Summary of our Design

Making a micro mouse requires two parts: hardware and software. 
Hardware
Our design uses following components: 
Sensors: Distance, Acceleration, Motor Speed, Heading, Magnetism
Computing: Raspberry Pi 5
Software: Python


## Summary of the MicroMouse Competition Rules

These are the rules for the IEEE micro mouse competition:

### Mouse
The following are not allowed: 
- Remote control.
- Combustion engines.
- Damaging, flying, jumping, climbing, marking the walls of the maze.
- Mouse larger than 25 cm in width or length.
- Change the code of the robot once maze is shown during competition.

### Maze
The maze is a 16x16 grid made of 18x18 (cm) squares. 
It has inside walls 5 (cm) high and 1.2 (cm) wide. 
The maze is built in such a way that robots that just follow walls cannot solve it. 
This forces us to use a refined approach .

### Teams
Teams are made of 5 people and cannot have more than two graduate students. 
Each team can use at most 3 robot designs but they cannot be exactly the same.

### Competition
Each robot starts in one of the four corners of the maze.  
The start cell is defined as a cell with a wall on 3 sides [1]. 
Teams are given 10min in total to solve the maze. 
A run starts when the robot crosses the startline and finishes when the robot crosses the entrance to the goal or when a participant touches it. 
The goal is located at the center of the maze and only has one entrance.
During the 10 minutes, teams can do multiple runs. 
A run is aborted by physically touching the mouse. 
Touching the mouse results in a 30 second malus. 
When a run is aborted the robot must re-start from the starting cell. 
The team with the shortest run time wins.
