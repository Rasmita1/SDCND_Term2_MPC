# SDCND_Term2_MPC
## Project Description
The purpose of this project is to develop a nonlinear model predictive controller (NMPC) to steer a car around a track in a simulator, and drive the car around the track. The simulator provides a feed of values containing the position of the car, its speed and heading direction. Additionally it provides the coordinates of waypoints along a reference trajectory that the car is to follow. All coordinates are provided in a global coordinate system. 

### The Vehicle Model
The vehicle model used in this project is a kinematic bicycle model. It neglects all dynamical effects such as inertia, friction and torque. The model takes changes of heading direction into account and is thus non-linear. The model used consists of the following equations
```
      // x_[t+1] = x[t] + v[t] * cos(psi[t]) * dt
      // y_[t+1] = y[t] + v[t] * sin(psi[t]) * dt
      // psi_[t+1] = psi[t] + v[t] / Lf * delta[t] * dt
      // v_[t+1] = v[t] + a[t] * dt
      // cte[t+1] = f(x[t]) - y[t] + v[t] * sin(epsi[t]) * dt
      // epsi[t+1] = psi[t] - psides[t] + v[t] * delta[t] / Lf * dt
```
Here, `x,y` denote the position of the car, `psi` the heading direction, `v` its velocity `cte` the cross-track error and `epsi` the orientation error. `Lf` is the distance between the center of mass of the vehicle and the front wheels and affects the maneuverability. The vehicle model can be found in the class `FG_eval`. 

### Polynomial Fitting and MPC Preprocessing
All computations are performed in the vehicle coordinate system. The coordinates of waypoints in vehicle coordinates are obtained by first shifting the origin to the current poistion of the vehicle and a subsequet 2D rotation to align the x-axis with the heading direction. Therby the waypoints are obtained in the frame of the vehicle. A third order polynomial is then fitted to the waypoints. The transformation between coordinate systems is implemented in `transformGlobalToLocal`. The transformation used is 
```
 X' =   cos(psi) * (ptsx[i] - x) + sin(psi) * (ptsy[i] - y);
 Y' =  -sin(psi) * (ptsx[i] - x) + cos(psi) * (ptsy[i] - y);  
```
where `X',Y'` denote coordinates in the vehicle coordinate system. Note that the initial position of the car and heading direction are always zero in this frame. Thus the state of the car in the vehicle cordinate system is 
```
          state << 0, 0, 0, v, cte, epsi;
```
initially. 


### Optimal Control Problem
For every state value provided by the simulator an optimal trajectory for the next N time steps is computed that minimizes a cost function. The cost function is quadratic in the cross-track error, the error in the heading direction, the difference to the reference velocity, the actuator values and the difference of actuator values in adjacent time steps. Specifically I chose parameter values here that lead to smooth driving both for slow (25mph) and fast velocities (70mph). The control problem is restricted by the vehicle model as well as actuator max and min values. The cost function can be found in `FG_eval`. 

In the receeding horizon problem the cost functio is minimized at each time step, but only the actuations corresponding to the first time step are sent to the simulator. At the next time step the entire optimal control problem is solved again.

### Model Predictive Control with Latency
An additional complication of this project consists in taking delayed actuations into account. After the solution of the NMPC problem a delay of 100ms is introduced before the actuations are sent back to the simulator. 

When delays are not properly accounted for oscillations and/or bad trajectories can occur. These delays make the control problem a so-called sampled NMPC problem.

Two common aproaches exist to take delays into account:
1. In one approach the prospective position of the car is estimated based on its current speed and heading direction by propagating the position of the car forward until the expected time when actuations are expected to have an effect. The NMPC trajectory is then determined by solving the control problem starting from that position. 
2. In the other approach the control problem is solved from the current position and time onwards. Latency is taken into account by constraining the controls to the values of the previous iteration for the duration of the latency. Thus the optimal trajectory is computed starting from the time after the latency period. This has the advantage that the dynamics during the latency period is still calculated according to the vehicle model. 
 
Here, I chose the second approach . The actuations are forced to remain at their previous values for the time of the latency. This is implemented in 
`MPC::Solve` like so. 

```  
  // constrain delta to be the previous control for the latency time
  for (int i = delta_start; i < delta_start + latency_ind; i++) {
    vars_lowerbound[i] = delta_prev;
    vars_upperbound[i] = delta_prev;
  }
 ... 
  
  // constrain a to be the previous control for the latency time 
  for (int i = a_start; i < a_start+latency_ind; i++) {
    vars_lowerbound[i] = a_prev;
    vars_upperbound[i] = a_prev;
  }
```

Accordingly, the value that is fed to the simulator is then taken to be the first freely varying control parameter of the optimal trajectory:
```
          // compute the optimal trajectory          
          Solution sol = mpc.Solve(state, coeffs);

          double steer_value = sol.Delta.at(latency_ind);
          double throttle_value= sol.A.at(latency_ind);
```
### Timestep Length and Elapsed Duration (N & dt)
The time `T=N dt` defines the prediction horizon. Short prediction horizons lead to more responsive controlers, but are less accurate and can suffer from instabilities when chosen too short. Long prediction horizons generally lead to smoother controls. For a given prediction horizon shorter time steps `dt` imply more accurate controls but also require a larger NMPC problem to be solved, thus increasing latency. 

Here I chose values of `N` and `dt` such that drives the car smoothly around the track for slow velocities of about 25mph all the way up to about 70mph. 
The values are `N=12` and `dt=0.05`.  Note that the `100ms = 2*dt` latency imply that the controls of the first two time steps are not used in the optimization. They are frozen to the values of the previous actuations, like so 

### Cost Function Parameters
The cost of a trajectory of length N is computed as follows
```
   Cost  = Sum_i cte(i)^2 
              + epsi(i)^2 
              + (v(i)-v_ref)^2 + delta(i)^2 
              + 10 a(i)^2 
              + 600 [delta(i+1)-delta(i)] 
              + [a(i+1)-a(i)]
```
where the increased weight on steering changes between adjacent time intervalls is the most important in order to arrive at smooth trajectories. 

I have used the codes present in the the sdcnd and referred the open forum portals.

## Dependencies

* cmake >= 3.5
* make >= 4.1
  * Linux: make is installed by default on most Linux distros
  * Mac: [install Xcode command line tools to get make](https://developer.apple.com/xcode/features/)
* gcc/g++ >= 5.4
  * Linux: gcc / g++ is installed by default on most Linux distros
  * Mac: same deal as make - [install Xcode command line tools]((https://developer.apple.com/xcode/features/)
  * Windows: recommend using [MinGW](http://www.mingw.org/)
* [uWebSockets](https://github.com/uWebSockets/uWebSockets) == 0.14, but the master branch will probably work just fine
  * Follow the instructions in the [uWebSockets README](https://github.com/uWebSockets/uWebSockets/blob/master/README.md) to get setup for your platform. You can download the zip of the appropriate version from the [releases page](https://github.com/uWebSockets/uWebSockets/releases). Here's a link to the [v0.14 zip](https://github.com/uWebSockets/uWebSockets/archive/v0.14.0.zip).
* [Ipopt](https://projects.coin-or.org/Ipopt)
  * Linux
    * You will need a version of Ipopt 3.12.1 or higher. The version available through `apt-get` is 3.11.x. If you can get that version to work great but if not there's a script `install_ipopt.sh` that will install Ipopt. You just need to download the source from the Ipopt [releases page](https://www.coin-or.org/download/source/Ipopt/) or the [Github releases](https://github.com/coin-or/Ipopt/releases) page.
    * Then call `install_ipopt.sh` with the source directory as the first argument, ex: `bash install_ipopt.sh Ipopt-3.12.1`. 
* [CppAD](https://www.coin-or.org/CppAD/)
  * Linux `sudo apt-get install cppad` or equivalent.
* [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page). This is already part of the repo so you shouldn't have to worry about it.
* Simulator. You can download these from the [releases tab](https://github.com/udacity/CarND-MPC-Project/releases).


## Basic Build Instructions
1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./mpc`.

I have not committed 'Eigen-3.3' folder as the size is big.
