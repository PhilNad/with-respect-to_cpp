# With-Respect-To (+ Expressed-In)
Simple library that manages databases of 3D transformations with explicit accessors.

## Goals
- Simple. A single 3D convention is used and is explicitly defined.
- Fast. The user should not worry about the overhead of calling accessors.
- Accessible. Information is accessible from a variety of interfaces on Linux (Python, Julia, Bash).
- Minimalist. The API contains as few methods as possible.

## Differences With tf2
Although this library might seem to be similar to [tf2](http://wiki.ros.org/tf2), there are notable differences that should be taken into account when choosing which one to use:
- ### **1.** tf2 requires running the ROS ecosystem, WRT does not.
  If you are already running ROS for other reasons, it might make more sense to continue using it. However, it might seem excessive to run the whole ROS ecosystem for the sole reason of recording transformations.
- ### **2.**  tf2 does transformation interpolation, WRT does not.
  One of the primary use case for tf2 is to linearly interpolate poses through time. WRT does not do such operation and assumes that all transformations are exact at the time at which they are accessed. If the previous pose of a moving frame must be recorded, a timestamp can be appended to the name of the frame to differentiate between the older and newer frames.
- ### **3** tf2 is designed as a distributed system while WRT is designed as a centralized system.
  While it is possible (and easy) to use WRT over ethernet through [a network file system](https://ubuntu.com/server/docs/service-nfs), it is certainly [not optimal](https://www.sqlite.org/useovernet.html). Conversely, while it is possible to use tf2 in a setup consisting in a single machine, it really shines when used by many machines over a network. However, be wary of the [bandwidth requirements](http://wiki.ros.org/tf2/Design#tf_messages_do_not_deal_with_low_bandwidth_networks_well) imposed by the frequent transfer of [ROS messages](https://docs.ros.org/en/lunar/api/geometry_msgs/html/msg/TransformStamped.html) over the network.

## Performances
Currently, a GET operation performed from within Bash takes about 0.009 seconds to execute while a SET operation from Bash takes about 0.04 seconds.
This is reasonable and allows any program that can run bash commands to use the interface.

## Design
- Uses the [Eigen library](https://eigen.tuxfamily.org)
- Produces and consumes 4x4 transformation Eigen matrices
- Store data in a SQLITE database using [sqlite3](https://docs.python.org/3/library/sqlite3.html)
- The scene is described by a tree
  - Re-setting a parent node, also changes the children nodes (i.e. assumes a rigid connection between parent and children)
  - If setting a transform would create a loop, the node is reassigned to a new parent. A frame only has a single parent.

## Dependencies
- [Python 3.x](https://www.python.org/downloads/)
- [Python 3.x Development Files](https://pkgs.org/download/python3-devel)
- [C++ 17 Compiler](https://gcc.gnu.org/) (Yes, you need C++17.)
- [CMake > 3.15](https://cmake.org/download/)
- [Eigen > 3.4](https://eigen.tuxfamily.org/)
- [pybind11](https://pybind11.readthedocs.io/en/stable/)
- [SQLiteCpp](http://srombauts.github.io/SQLiteCpp/)
- [sqlite3](https://sqlite.org/index.html)

### Dependencies Installation
Most dependencies should be provided by your preferred package manager. PyBind11 and SQLiteCpp are included in this repository as submodules and can be easily installed with:
```bash
> git submodule update --init --recursive
> cd extern/pybind11
> cmake -S . -B build
> cmake --build build
> sudo cmake --install build
> cd ../SQLiteCpp
> cmake -S . -B build
> cmake --build build
> sudo cmake --install build
```

## Installation From Pre-Compiled Binaries
The [latest release](https://github.com/PhilNad/with-respect-to/releases) contains pre-compiled 64-bit binaries for Linux, that can be used directly once placed in an appropriate directory.

## Installation From Source Code
### Build Executables
```bash
> git clone https://github.com/PhilNad/with-respect-to.git
> cd with-respect-to
> cmake -S . -B build
> cmake --build build
```
### Install Everything
```bash
> sudo cmake --install build
```

### Install Command-Line Interface (cli)
```bash
> cd cli
> sudo make install
```
### Install Python3 bindings
```bash
> cd ../python_bindings
> sudo make install
```

### Installation Verification
```bash
> cd ~
> WRT
> python3 -c $'import with_respect_to as WRT'
```

## Usage of the Command-Line Interface
```
Usage: WRT [options] 

Optional arguments:
-h --help    	shows help message and exits
-v --version 	prints version information and exits
-q --quiet   	If a problem arise, do now output any information, fails quietly. [default: false]
-c --compact 	Output a compact representation of the matrix as a comma separated list of 16 numbers in row-major order. [default: false]
--In         	The world name the frame lives in ([a-z][0-9]-). [required]
--Get        	Name of the frame to get ([a-z][0-9]-).
--Set        	Name of the frame to set ([a-z][0-9]-).
--Wrt        	Name of the reference frame the frame is described with respect to ([a-z][0-9]-). [required]
--Ei         	Name of the reference frame the frame is expressed in ([a-z][0-9]-). [required]
--As         	If setting a frame, a string representation of the array defining the pose with rotation R and translation t: [[R00,R01,R02,t0],[R10,R11,R12,t1],[R20,R21,R22,t2],[0,0,0,1]]
```

### Example Usage From Bash
```bash
> WRT --In test --Get d --Wrt a --Ei a
 0 -1  0  1
 0  0 -1  0
 1  0  0  1
 0  0  0  1
> WRT --compact --In test --Get d --Wrt a --Ei a
0,-1,0,1,0,0,-1,0,1,0,0,1,0,0,0,1
> WRT --In test --Set a --Wrt world --Ei world --As [[1,0,0,1],[0,1,0,1],[0,0,1,1],[0,0,0,0]]
The format of the submitted matrix is wrong (-3).
> WRT --quiet --In test --Set a --Wrt world --Ei world --As [[1,0,0,1],[0,1,0,1],[0,0,1,1],[0,0,0,0]]
> WRT --In test --Set a --Wrt world --Ei world --As [[1,0,0,1],[0,1,0,1],[0,0,1,1],[0,0,0,1]]
> WRT --dir /home/username/other_dir/ --In test --Get d --Wrt a --Ei a
The reference frame a does not exist in this world.
```

### Example Usage From C++
See [test/src/test.cpp](test/src/test.cpp).

### Example Usage From Python
See [python_bindings/src/test.py](python_bindings/src/test.py).

## Debugging
You can use [SQLiteStudio](https://github.com/pawelsalawa/sqlitestudio) to open the database file and read its content through a GUI.

## TODO
- [x] ~Allow saving to memory instead of file to avoid cluttering the space with temporary databases~
  - Using the sqlite3 feature (with :memory: filename) of working in memory is difficult to use with SQLiteCpp as the interface makes it hard to save a reference to a database for later use. Instead, we are now enabling the user to pass a flag to the library such that the database file will be deleted upon destruction of the database object.
- [x] Allow setting a pose wrt a reference that does not exist yet.
- [x] Allow the tree of reference frames to be disconnected from the 'world' frame.
- [x] Use quaternions under the hood to avoid returning unorthogonal matrices after lots of compositions.
- [ ] Use the shortest path between reference frames in the tree (is it worth it?)
- [ ] Make it possible to have loops in the pose graph (tf cannot do that), and it would be useful for closed kinematic chains
- [ ] Test that using this library from multiple scripts produces the intended results.
- [ ] Make Julia bindings to the library.
- [ ] Better documentation of the library.
- [x] ~Remove files related to SQLiteCpp from the repository.~
  - Now SQLiteCpp is a submodule in ./extern/
- [x] Make a x64 Linux executables package for easy installation.
