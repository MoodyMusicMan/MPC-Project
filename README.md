# Model Predictive Controller


## Descriton
The purpose of this program is to implement a model predictive controller to control the
actuators of a simulated car, mainly the Steering and Acceleration.

This program works in unison with a simulator, which you can find [here](https://github.com/udacity/self-driving-car-sim/releases).


This project was completed as part of the UDACITY Self-Driving Car Engineer Nanodegree Program. For more information, or 
to enroll today, please check [here.](https://www.udacity.com/course/self-driving-car-engineer-nanodegree--nd013) 


## The Model

### Establish Sate and Waypoints
The model uses a vehicle state vector in combination with a set of way points in order to predict
the actuations that should be taken by the vehicle.

The state vector is comprised of the P (x, y position), V (velocity), Psi (Bearing), CTE (Distance from Waypoint Polynomial), and ePsi (Bearing difference 
compared to polynomial).

The waypoints are provided by the simulator (assumedly from Computer vision, Particle Filtering + mapping, or some other means).
These waypoints are converted from map coordinates to vehicle coordinates, and then turned from a discrete set of values, to a continous polynomial by our provided function, polyfit(). The waypoint polynomial is used to calulate CTE and ePsi

Using our wapoint polynomial, we calculate 25 waypoints over 75m to be displayed in our simulator. This is done by
 sending the inputs to polyeval() to our next_x_vals and the returns to our next_y_vals. The simulator displays the
 values contained in next_x and next_y (with repsect to the car).
 
### Determine Actuations

#### Solve()
We use our state and polynomial calculated in the prior step as an input to our `Solve()` function. `Solve()` shapes an array
to match the state vector, actuation values, and a predetermined number of future states **N** (more on N later). Each state and actuaion value
comprises N number of elements in this vector, with the first element of each variable relfecting the current state, and each element
thereafter representing an incremental future state. `Solve()` then polulates this array with minimum and maximum values for each variable, 
such as maximum turning angle of +/- 25 degrees, or maximum acceleration of +/- 1. Finally, the array is initialized with 
the current state, and that state is locked so that it cannot be changed by our evaluator. `Solve()` then calls `fg_eval()`

#### Evaluate
`fg_eval` is preloaded with a **cost** function, which associates a certain cost with the state of our vehicle. The cost 
of each state is determined by multiplying the state (or group of states) with a weight. Highly weighted states
will be very costly, and low values will be less costly.

For example:
* 1500 * CTE^2  : reflects that high CTE values will be very costly
* 1250*(delta(t = 0) - delta(t = 1))^2 : reflects that abrupt changes in steering angle will be very costly
* 5*acceleration^2 : reflects that acceleration and deceleration will not cost very much.

`fg_eval` then establishes a set of state equations, that reflects how each state will change over time with regards to
each other state variable, and any actuator variables.

These state equations, cost functions, and current state are fed into an optomizing function, `IPOPT`.

#### IPOPT

IPOPT uses the current state, cost functions, and state equations, N future states, and a time step dt (more on dt later) to calculate the minimum cost value possible. The state
vector consists of dependant variables, while the actuators serve as independent variables. This means that the optomizer 
can only affect the vehicle state (this a large portion of the cost) through the actuators.

The output of our IPOPT calculations are returned to our `Sovle()` Function by means of the array we formatted earlier.
The acceleration and turning value for our vehicle are extracted from this array, along with the position measurements predicted 
our optimizer. `Solve()` finally returns these values back to the main file. Where the main file implements the Acceleration
/Turning commands, and plots the predicted x/y coordinates in Green.


## Selection of N and dt

`dt` is the amount of time between each prediction, and `N` is the number of predictions to make. `dt` was chosen to 
be .175s in order accomdate the inability of the vehicle to make actuation adjustments faster than once every 100 ms. `N`
was chosen to be 10 in order to give a reasonable glimpse into the future (1.7 seconds), while still being low enough to 
prevent the cost evaluation from making strange decisions.


## Controls Latency

The 100ms latency of the controls was combated through 2 measures. The first was setting `dt` the time step between predicted
states to a value larger than the latency. This would cause the predictions to  acomadate the delay in controls, and any messaging
between the simulator and program.

The second, and more effective method was predicting the state of the vehicle after 100ms, and using that as the state input
to our `Solve()` function. This was accomplished using basic position equations, and an estimated scaler acceleration with
max acceleration at 10mph/s (measured by eyeball).

**Note** - The equations for predicting the future state of the vehicle could use a little tweaking, however they appear to be asound strategy
for dealing with Latency. Setting the dt to a large value however, does not seem like a sound strategy, as it causes turning 
to be clunky, and results in much less granular control of the vehicle. This could be dangerous in some instances. It is possible 
that calculating a large time latency, and a short time latency prediction (2 predictions), might be a strategy to investigate
for future developments. This would allow the large time delay to set the general path, and the small time delay to attemp to
further optimize the vehicles manuevers. Additionally, limiting the actuators to only be able to act once every latency unit 
would be a strategy worth investigating. This could be done by setting the maximum/minimum values in our f_g function, to be
the previous actuation command, except where a latency timestep fell, where an actuation command would be set to max/min control bounds.
This is mostly speculation though, and requires further efforts.


## Problems/ Known Bugs

Although the program compiles, and is able to drive around the track successfully, there are some known issues with the code

* Predicted Path (green) diverges from polynomial at far away waypoints. It is speculated that this is due to our predicted
future state surpassing our waypoints, causing our cost fucntion to make strange decisions at outer time steps.
* Slow Turning, likely caused by a high cost associated with change in turning over time. This could likely be solved by 
adjusting weight values associated with turn rate, and change in turn rate. Preliminary tests have caused vehicle to verge 
off the road, however further investigation will likely prove fruitful.






## Dependencies


* cmake >= 3.5
 * All OSes: [click here for installation instructions](https://cmake.org/install/)
* make >= 4.1(mac, linux), 3.81(Windows)
  * Linux: make is installed by default on most Linux distros
  * Mac: [install Xcode command line tools to get make](https://developer.apple.com/xcode/features/)
  * Windows: [Click here for installation instructions](http://gnuwin32.sourceforge.net/packages/make.htm)
* gcc/g++ >= 5.4
  * Linux: gcc / g++ is installed by default on most Linux distros
  * Mac: same deal as make - [install Xcode command line tools]((https://developer.apple.com/xcode/features/)
  * Windows: recommend using [MinGW](http://www.mingw.org/)
* [uWebSockets](https://github.com/uWebSockets/uWebSockets)
  * Run either `install-mac.sh` or `install-ubuntu.sh`.
  * If you install from source, checkout to commit `e94b6e1`, i.e.
    ```
    git clone https://github.com/uWebSockets/uWebSockets 
    cd uWebSockets
    git checkout e94b6e1
    ```
    Some function signatures have changed in v0.14.x. See [this PR](https://github.com/udacity/CarND-MPC-Project/pull/3) for more details.
* Fortran Compiler
  * Mac: `brew install gcc` (might not be required)
  * Linux: `sudo apt-get install gfortran`. Additionall you have also have to install gcc and g++, `sudo apt-get install gcc g++`. Look in [this Dockerfile](https://github.com/udacity/CarND-MPC-Quizzes/blob/master/Dockerfile) for more info.
* [Ipopt](https://projects.coin-or.org/Ipopt)
  * If challenges to installation are encountered (install script fails).  Please review this thread for tips on installing Ipopt.
  * Mac: `brew install ipopt`
       +  Some Mac users have experienced the following error:
       ```
       Listening to port 4567
       Connected!!!
       mpc(4561,0x7ffff1eed3c0) malloc: *** error for object 0x7f911e007600: incorrect checksum for freed object
       - object was probably modified after being freed.
       *** set a breakpoint in malloc_error_break to debug
       ```
       This error has been resolved by updrading ipopt with
       ```brew upgrade ipopt --with-openblas```
       per this [forum post](https://discussions.udacity.com/t/incorrect-checksum-for-freed-object/313433/19).
  * Linux
    * You will need a version of Ipopt 3.12.1 or higher. The version available through `apt-get` is 3.11.x. If you can get that version to work great but if not there's a script `install_ipopt.sh` that will install Ipopt. You just need to download the source from the Ipopt [releases page](https://www.coin-or.org/download/source/Ipopt/).
    * Then call `install_ipopt.sh` with the source directory as the first argument, ex: `sudo bash install_ipopt.sh Ipopt-3.12.1`. 
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [CppAD](https://www.coin-or.org/CppAD/)
  * Mac: `brew install cppad`
  * Linux `sudo apt-get install cppad` or equivalent.
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page). This is already part of the repo so you shouldn't have to worry about it.
* Simulator. You can download these from the [releases tab](https://github.com/udacity/self-driving-car-sim/releases).
* Not a dependency but read the [DATA.md](./DATA.md) for a description of the data sent back from the simulator.


## Basic Build Instructions

1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./mpc`.
