# libMobility

Documentation is available at [](libmobility.readthedocs.io)  
This repository contains several GPU solvers that can compute the action of the hydrodynamic mobility (at the RPY/FCM level) of a group of particles (in different geometries) with forces acting on them.  

In particular, given a group of forces, $\boldsymbol{F}$ , acting on a group of positions, $\boldsymbol{X}$, the libMobility solvers can compute:

$$d\boldsymbol{X} = \boldsymbol{\mathcal{M}}\boldsymbol{F} + \text{prefactor}\sqrt{2T\boldsymbol{\mathcal{M}}}d\boldsymbol{W}$$  

<!--- Donev: The default value for prefactor is 1. Default value should be 0, and in that case the calculation of sqrtW should be omitted and zero returned for efficiency. ---> 

Where dX are the linear displacements, prefactor is an user provided prefactor and dW is a collection of i.i.d Weinner processes. T is the temperature (really $k_B T$). Finally $\boldsymbol{\mathcal{M}}$ represents the mobility tensor.  
Each solver in libMobility allows to compute either the deterministic term, the stochastic term, or both at the same time.  

For each solver, a python and a C++ interface are provided. All solvers have the same interface, although some input parameters might change (an open boundaries solver does not accept a box size as a parameter).  

Some of the solvers have different functionalities than the others. For instance, some modules accept torques while others don't. The documentation for a particular solver must be visited to know more.  

## How this repository is organized

The directory **solvers** contains a subfolder for each module. Each module exists on a different git repository (and included here as a submodule). Besides the submodule repo, the folder for each solver contains a wrapper to the afforementioned repository that allows to use it under a common interface.  
The directory **python** contains the file libMobility.py, which can be imported giving access to all the compiled modules.  
The directory **cpp** contains an example usage of the C++ interface to the modules.  
The directory **include** contains the C++ base class and utilities used to construct the modules.  

## The libMobility interface
Each solver is encased in a single class which is default constructible (no arguments required for its constructor).  
Each solver provides the following set of functions (called the same in C++ and python and described here in a kind of language agnostic way):  
  * **[Constructor] (configuration)**: The solver constructors must be provided with a series of system-related parameters (see below).  
  * **initialize(parameters)**: Initializes the module according to the parameters (see below).  
  * **setParameters[SolverName]([extra parameters])**: Some modules might need special parameters, in these instances this function must also be called. Check the README for each module and its mobility.h file.  
  * **setPositions(positions)**: Sets the positions to compute the mobility of.  
  * **Mdot(forces, result)**: Computes the deterministic hydrodynamic displacements, i.e applies the mobility operator. 
  * **sqrtMdotW(result, prefactor = 1)**: Computes the stochastic displacements and multiplies them by the provided prefactor.
!--- Donev: If prefactor=0 do nothing --->    
  * **hydrodynamicVelocities(forces = null, torques = null, result, prefactor = 1)**: Equivalent to calling Mdot followed by sqrtMdotW (some algorithms might benefit from doing these operations together, e.g., solvers based on fluctuating hydrodynamics).  
!--- Donev: default value of prefactor should be 0. This function is confusing with regards to the torques. NOTE: I only later saw torques were removed from mobility.h... I assume what you mean here is that stochastic increments are only computed for linear velocity, and if torques are provided only the deterministic term is provided? If you mean that M in the case of torques is the combined force-torque mobility, then forces and torques should be in one vector called "applied". There has to be consistency between result and input and cleared up. I have a strong feeling I wrote this comment already but not sure you ever saw it or never looked at my last round of comments.  --->  
  * **clean()**: Cleans any memory allocated by the module. The initialization function must be called again in order to use the module again.  
The many examples in this repository offer more insight about the interface and how to use them. See cpp/example.cpp or python/example.py. See solvers/NBody for an example of a module implementation. Even though the algorithm behind it is quite convoluted, the files in this directory are short and simple, since they are only a thin wrapper to the actual algorithm, located under BatchedNBodyRPY there.  
An equal sign denotes defaults.  

### Data format
Positions, forces, torques and the results provided by the functions are packed in a 3*numberParticles contiguos array containing ```[x_1, y_1, z_1, x_2,...z_N]``` .

!--- Donev: This is not clear if both forces and torques are provided as input. What is the format of result? 6*N or 2*3*N? --->  

### Parameters
The valid parameters accepted by the interface are:  
  * **temperature**. In units of energy (AKA k_BT).  
  * **hydrodynamicRadius**: The hydrodynamic radii of the particles. Note that many solvers only allow for all particles having the same radius, in those cases this vector should be of size one.  
  * **viscosity**: The fluid viscosity.  
  * **tolerance = 1e-4**: Tolerance for the Lanczos algorithm.  
  * **numberParticles**: The number of particles  

An equal sign denotes default values.  

### Configuration parameters
At contruction, solvers must be provided with the following information:
  * **periodicityX**, **periodicityY**, **periodicityZ**: The periodicity, can any of "periodic", "open", "single_wall", "two_walls", "unspecified".  
  
The solvers constructor will check the provided configuration and throw an error if something invalid is requested of it (for instance, the PSE solver will complain if open boundaries are chosen).


## How to use this repo
Some solvers might be included as a git submodule (So that each solver has its own, separated, repository). Be sure to clone this repository recursively (using ```git clone --recurse```).  

After compilation (see below) you will have all the tools mentioned above available for each solver. Note that it is not required to compile all modules, only the ones you need (so you do not need to worry about dependencies of unused solvers).

## Compilation

We recommend working with a [conda](https://docs.conda.io/en/latest/) environment. The file environment.yml contains the necessary dependencies to compile and use the library.

You can create the environment with:

```shell
$ conda env create -f environment.yml
```

Then, activate the environment with:

```shell
$ conda activate libmobility
```

CMake is used for compilation, you can compile and install everything with:

```shell
$ mkdir build && cd build
$ cmake -DCMAKE_INSTALL_PREFIX=$CONDA_PREFIX ..
$ make all install
```
It is advisable to install the library in the conda environment, so that the python bindings are available. The environment variable $CONDA_PREFIX is set to the root of the conda environment.  

CMake will compile all modules under the solvers directory as long as they adhere to the conventions described in "Adding a new solver".  

After compilation, the python bindings will be available in the conda environment under the name libMobility.  

The following variables are available to customize the compilation process:  

  * DOUBLEPRECISION : If this variable is defined libMobility is compiled in double precision (single by default).  


## Python Usage

Importing libMobility will make available any module under "solvers".  

An usage example is available in python/example.py.  
Calling
```python
	help(SolverName)
```
will provide more in depth information about the solver.  

## C++ Usage

In order to use a module called SolverName, the header solvers/SolverName/mobility.h must be included.  
If the module has been compiled correctly the definitions required for the functions in mobility.h will be available at solvers/SolverName/mobility.so.  
An example is available in cpp/example.cpp and the accompanying Makefile.  
## Adding a new solver

Solvers must be added following the ```libmobility::Mobility``` directives (see include/MobilityInterface). This C++ base class joins every solver under a common interface. When the C++ interface is prepared, the python bindings can be added automagically by using ```pythonify.h``` (a tool under MobilityInterface).  

Solvers are exposed to Python and C++ via a single class that inherits from ```libmobility::Mobility```.  

A new solver must add a directory under the "solvers", which must be the name that both the python and C++ classes.  

Under the directory proper to this new module, the repository containing the solver can be added as a submodule.

Besides the folder containing the repository, inside the solver directory a series of files (and only these files) must exist (additional files might be placed under an "extra" directory inside the solver directory). These files are:  
  * **mobility.h**: Must include ```MobilityInterface.h``` and define a single class, named as the directory for the solver, that inherits the ```libmobility::Mobility``` base class. For instance, in solvers/NBody/mobility.h we have:  
  ```c++
  ...
  class NBody: public libmobility::Mobility{
  ...
  ```
Every pure virtual function in ```libmobility::Mobility``` must be overriden and defined. See more about the mobility interface in include/MobilityInterface.  
mobility.h can include and make use of any existent utilities/files from the solvers repository.  
  * **python_wrapper.cpp**: This file must provide the python bindings for the class in mobility.h. If this class follows the ```libmobility::Mobility``` correctly, this file can in general be quite simple, having only a single line using the ```MOBILITY_PYTHONIFY``` utility in include/MobilityInterface/pythonify.h. See solvers/NBody/python_wrapper.cpp for an example.  
  In the case of a module being python only (or in general not providing a correct child of ```libmobility::Mobility```), python_wrapper.cpp might be ommited, but a file called [solver].py must exist instead, providing a python class that is compatible with ```libmobility::Mobility``` (so that the user can write ```from solver import *``` and get a class, called "solver" that adheres to the libmobility requirements).  
  * **CMakeLists.txt**: This must contain rules to create the shared library for the particular solver and its python wrapper. The solver library should be called "lib[Solver].so", while the python library should be called "[Solver].[Python_SOABI].so" with the correct extension suffix. See one of the available CMakeLists.txt for an example.
 * **example.py**: An example script using the python bindings of the module.  
 * **test.py**: A correctness test (as simple as possible) that ensures the module is working as intended.  
 * **README.md**: The documentation for the specific module (stating the geometry, required arguments,...).  

Finally, a new line should be added to solvers/CMakeLists.txt to include the new module in the compilation process.

Regarding the different functions of the interface, some of them provide default behavior if not defined. In particular, the stochastich displacements will be computed using a Lanczos solver if the module does not override the corresponding function. Additionally, the hydrodynamicDisplacements functions defaults to calling Mdot followed by stochasticDisplacements. Finally, the clean function defaults to doing nothing.  
An example of this is NBody, which only provides an initialization and Mdot functions.  

**The initialize function of a new solver must call the ```libmobility::Mobility::initialize``` function at some point.**  

**See solvers/SelfMobility for a basic example.**

When a module needs additional parameters to those provided to initialize an additional function, called ```setParameters[SolverName]``` must be defined and exposed to python. See solvers/PSE/mobility.h and solver/PSE/python_wrapper.cpp for an example. It is up to users of the library to call setParameters before calling initialize with the required arguments.

