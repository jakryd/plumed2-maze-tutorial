##### [&larr; Home](NAVIGATION.md)

### Input

Files needed to complete this tutorial are provided in the `data` directory available in the [GitHub repository](https://github.com/jakryd/plumed2-maze-tutorial).

#### Groups

We first define the groups of atoms consisting of T4L (atoms 1-2634) and benzene (atoms 2635-2646) and their center of masses. These definitions will be passed to an optimizer later. 

```plumed
GROUP ATOMS=2635-2646 LABEL=group_bnz
GROUP ATOMS=1-2634 LABEL=group_t4l

CENTER ATOMS=group_bnz LABEL=center_bnz
CENTER ATOMS=group_t4l LABEL=center_t4l
```

#### Optimizer and Loss Function

Next, we declare an optimizer that will find a suboptimal (in terms of loss function) unbinding direction for the ligand. For this purpose, we will use the simulated annealing algorithm (`MAZE_OPT_ANNEALING`). The optimization will be run during the MD simulation every `STRIDE` MD steps (here 1 ns) with 1000 iterations to converge (`N_ITER`).

```plumed
#HIDDEN
GROUP ATOMS=2635-2646 LABEL=group_bnz
GROUP ATOMS=1-2634 LABEL=group_t4l

CENTER ATOMS=group_bnz LABEL=center_bnz
CENTER ATOMS=group_t4l LABEL=center_t4l
#ENDHIDDEN
MAZE_OPT_ANNEALING ...
  LABEL=opt

  GROUPA=group_bnz
  GROUPB=group_t4l

  SWITCH={EXP R_0=0.2}

  STRIDE=500000

  N_ITER=1000

  BETA=0.9
  BETA_FACTOR=1.005
  BETA_SCHEDULE=GEOM

  RANDOM_SEED=111
  
  NLIST
  NL_CUTOFF=0.6
  NL_STRIDE=100
...
```

The `BETA` options are required to select an annealing scheme. Here, we will start with an inverse temperature parameter $\beta=0.9$ multiplied by a factor of 1.005 every iteration (`BETA_SCHEDULE=GEOM`). This is a standard setup -- interested readers can read more about it [here](https://en.wikipedia.org/wiki/Simulated_annealing)).

To speed up calculations, we will use a neighbor list `NLIST` with a cutoff of 0.6 nm for distances between ligand-protein atom pairs (see [PLUMED Neighbor Lists](https://www.plumed.org/doc-v2.9/user-doc/html/_neighbour.html)). The neighbor list will be recalculated every 100 steps (`NL_STRIDE`).

The `SWITCH` keyword specifies the loss function to optimize. The same functionality is provided for switching functions required to define collective variables such as coordination numbers. See [here](https://www.plumed.org/doc-v2.9/user-doc/html/switchingfunction.html) for switching functions available in PLUMED. Here, we will use an exponential loss function as explained in [Background](background.md).

#### Bias

Having defined the optimizer, we will pass it to the adaptive bias potential (see [here](background.md#adaptive-biasing)). The bias will be used to steer benzene in the optimal direction found by simulated annealing. This also requires a collective variable with Cartesian components to drive the ligand toward solvent. 

*Note* Biasing absolute positions of any system can result in problems (see a detailed explanation [here](https://www.plumed.org/doc-v2.9/user-doc/html/_p_o_s_i_t_i_o_n.html)) and may require additional position restraints. Thus, it is required that it must be a distance with `COMPONENTS`. As such, the biasing corresponds to a relative position, and additional restraints are not required. 

The `HEIGHT` and `RATE` keywords give the scale constant and rate, respectively.

```plumed
#HIDDEN
GROUP ATOMS=2635-2646 LABEL=group_bnz
GROUP ATOMS=1-2634 LABEL=group_t4l

CENTER ATOMS=group_bnz LABEL=center_bnz
CENTER ATOMS=group_t4l LABEL=center_t4l

MAZE_OPT_ANNEALING ...
  LABEL=opt

  GROUPA=group_bnz
  GROUPB=group_t4l

  SWITCH={EXP R_0=0.2}

  STRIDE=500000

  N_ITER=1000

  BETA=0.9
  BETA_FACTOR=1.005
  BETA_SCHEDULE=GEOM

  RANDOM_SEED=111
  
  NLIST
  NL_CUTOFF=0.6
  NL_STRIDE=100
...
#ENDHIDDEN
DISTANCE ATOMS=center_bnz,center_t4l LABEL=dist COMPONENTS

MAZE_OPT_BIAS ...
  LABEL=bias

  ARG=dist.x,dist.y,dist.z

  HEIGHT=1000.0

  RATE=0.001

  OPTIMIZER=opt
...
```

We will also define a committor indicator to halt the simulation once benzene enters the solvent, which occurs approximately when the distance between the ligand and the protein exceeds 3 nm.

```plumed
#HIDDEN
GROUP ATOMS=2635-2646 LABEL=group_bnz
GROUP ATOMS=1-2634 LABEL=group_t4l

CENTER ATOMS=group_bnz LABEL=center_bnz
CENTER ATOMS=group_t4l LABEL=center_t4l
#ENDHIDDEN
DISTANCE ATOMS=center_bnz,center_t4l LABEL=dist_d

COMMITTOR ...
  ARG=dist_d
  STRIDE=5000
  BASIN_LL1=3
  BASIN_UL1=10
...
```

#### Simulation

We are ready to run the simulation.

The simulation will save two files: `colvar.dat` and `optimizer.dat`. The `colvar.dat` file contains all the information we want to save using the `PRINT` action, while the `optimizer.dat` file contains data related to the optimization results (see [here](https://github.com/jakryd/plumed2-maze-tutorial/tree/main/data)).

##### [Results &rarr;](results.md)