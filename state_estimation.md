# State Estimation

One-line memory hook:

**Like returning a tennis shot: read your opponent's hit, ball condition, and early trajectory, then decide where to move and how to strike from the best current estimate.**

Major components of the problem:

1. **State model**: represents the internal state we care about.
2. **Observations**: noisy/partial measurements from sensors.
3. **Update rule**: use observations to tweak state so model and data align better.
4. **Prediction**: use the updated state/model to predict what happens next.

State estimation vs ML (simple view):

- **State estimation**: model-based mathematical inference (rules/equations + uncertainty).
- **ML**: data-driven function learning from examples.
- **In practice**: often combined (ML for perception, state estimation for fusion/tracking).

What GTSAM is:

- **GTSAM** is a modeling + solving library for state estimation.
- It represents the problem as a mathematical abstraction (states, measurements, uncertainty) using factor graphs.
- It provides optimization/inference tools to solve that representation and produce estimates.

Hello world problem (task-driven wording):

- Task: move a robot from point A to point B, step by step.
- At each step, we must decide the next motion (rotation + forward progress).
- To make that decision, we need a good estimate of the robot pose (position + orientation).
- We only observe pose indirectly from wheel odometry and weak GPS-like signals, and both are noisy.
- Core question: how do we combine these weak observations to get a better pose estimate so the robot can move to the target reliably?

How to model this in GTSAM (walkthrough):

1. **Define states**: pose at each step is a state variable (`x0, x1, x2, ...`) and is the target we want to estimate.
2. **Define factors (constraints)** that inform those states:
   - wheel odometry factors between consecutive poses
   - GPS-like factors that anchor some poses to absolute observations
   - an initial pose prior factor to anchor the full graph
3. **Run an optimizer** over the whole factor graph to compute the best pose estimates from all noisy constraints.

Offline vs online in GTSAM:

- **Offline (batch)**: we have all measurements from a run, then optimize together.
- **Online (incremental/near real-time)**: we add new measurements over time and update the estimate incrementally.
- This walkthrough is the **offline** style.
- Important: in offline mode, all **measurements/factors** are available; poses in `initialEstimate` are still rough guesses that the optimizer refines.

States (poses) in this example:

- Each step has a pose state (`x, y, theta`).
- The pose values in `initialEstimate` are a **best effort** guess during data collection (or simple preprocessing), not final truth.
- We optimize these initial pose values using all factors to get better final estimates.

```cpp
Values initialEstimate;
initialEstimate.insert(1, Pose2(0.5, 0.0, 0.2));
initialEstimate.insert(2, Pose2(2.3, 0.1, -0.2));
initialEstimate.insert(3, Pose2(4.1, 0.1, M_PI_2));
initialEstimate.insert(4, Pose2(4.0, 2.0, M_PI));
initialEstimate.insert(5, Pose2(2.1, 2.1, -M_PI_2));
```

Factors (observations/constraints) in this example:

- Factors are how observations and constraints enter the graph.
- Wheel odometry becomes `BetweenFactor<Pose2>` between consecutive poses.
- GPS-like absolute observations are often modeled as prior-like factors on specific poses.
- This example includes a start prior + odometry + loop-closure factor (loop closure is also an observation-derived constraint).

```cpp
// Prior on first pose (anchor)
auto priorNoise = noiseModel::Diagonal::Sigmas(Vector3(0.3, 0.3, 0.1));
graph.addPrior(1, Pose2(0, 0, 0), priorNoise);

// Odometry factors (relative motion)
auto model = noiseModel::Diagonal::Sigmas(Vector3(0.2, 0.2, 0.1));
graph.emplace_shared<BetweenFactor<Pose2> >(1, 2, Pose2(2, 0, 0), model);
graph.emplace_shared<BetweenFactor<Pose2> >(2, 3, Pose2(2, 0, M_PI_2), model);
graph.emplace_shared<BetweenFactor<Pose2> >(3, 4, Pose2(2, 0, M_PI_2), model);
graph.emplace_shared<BetweenFactor<Pose2> >(4, 5, Pose2(2, 0, M_PI_2), model);

// Loop closure factor
graph.emplace_shared<BetweenFactor<Pose2> >(5, 2, Pose2(2, 0, M_PI_2), model);
```

Loop closure, in plain language:

- In this example, pose `5` revisits the area related to earlier pose `2`.
- Because odometry drifts, these poses may not align perfectly in the initial estimate.
- A loop-closure factor adds a measured **relative pose** constraint between `5` and `2`.
- Relative pose means translation + rotation (`dx, dy, dtheta`), not just Euclidean distance.
- Optimization uses this extra constraint to reduce drift and improve global consistency of all poses.

Optimizer cheat sheet (GN vs LM):

- **Gauss-Newton (GN)**: faster near a good initial guess, but less robust when initialization is poor.
- **Levenberg-Marquardt (LM)**: more robust with poor initial guesses, usually a bit more conservative.
- Practical rule: start with **LM** for stability; switch to **GN** when initialization is already good and speed matters.
- In incremental/online settings, GTSAM often uses **iSAM2** (incremental updates) instead of rerunning a full batch optimizer each time.

Optimizer details (GN vs LM):

Gauss-Newton
- Uses a local quadratic approximation and takes a full step toward the minimum.
- Usually faster when your initial guess is already close.
- Can struggle/diverge if initial guess is poor or problem is highly nonlinear.
- Think: aggressive and efficient near the solution.

Levenberg-Marquardt
- Blends Gauss-Newton with gradient-descent-like damping.
- More robust when initial guess is not great.
- Typically a bit more conservative/slower per progress than GN.
- Think: safer and more stable on tough starts.

How LM optimizer is added in code:

```cpp
#include <gtsam/nonlinear/LevenbergMarquardtOptimizer.h>

LevenbergMarquardtParams params;
params.maxIterations = 100;
params.relativeErrorTol = 1e-5;

LevenbergMarquardtOptimizer optimizer(graph, initialEstimate, params);
Values result = optimizer.optimize();
```

What each line means:

- `LevenbergMarquardtParams params;` creates solver settings ("knobs").
- `maxIterations` sets a hard cap on how many update steps LM can run.
- `relativeErrorTol` sets a stop condition when improvement becomes very small.
- `LevenbergMarquardtOptimizer(...)` builds the optimizer with graph + initial guesses + settings.
- `optimizer.optimize()` runs iterative refinement and returns optimized states in `result`.

Example reference + steps to run:

- Example file: https://github.com/borglab/gtsam/blob/develop/examples/Pose2SLAMExample.cpp

1. From repo root, create and enter build directory:
   - `mkdir -p build && cd build`
2. Configure with CMake:
   - `cmake ..`
3. Build and run the example target:
   - `make Pose2SLAMExample.run`
4. Check output sections:
   - `Initial Estimate` (best-effort starting poses)
   - `Final Result` (optimized poses)
   - covariance printout (`x1 covariance`, etc.)



