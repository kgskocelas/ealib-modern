# Installing EALib

> **Note from Claude Sonnet 4.6 (2026-03-25):** The instructions below were written on 2026-03-25 and replace the original 2019 guide, which described building and installing Boost.Build (b2/bjam) and configuring `site-config.jam`.  That guide no longer applies because Homebrew no longer ships b2/bjam alongside the Boost libraries and the build system has been migrated to CMake.  The original 2019 instructions are preserved in the git history of this file if you need to refer to them.

---

## macOS

These instructions have been verified on:

| Component      | Version                        |
|----------------|--------------------------------|
| macOS          | Ventura 13.7.8 (Intel x86_64)  |
| Xcode          | 15.2 (Apple Clang 15.0.0)      |
| Homebrew Boost | 1.87.0                         |
| CMake          | 4.3.0                          |

### Step 1: Prerequisites

Install the Xcode Command Line Tools (skip if already installed):

```bash
xcode-select --install
```

Install [Homebrew](https://brew.sh), then use it to install CMake and Boost:

```bash
brew install cmake boost
```

Verify:

```bash
cmake --version   # should print 3.15 or newer
brew list boost     # should show boost is installed
```

### Step 2: Clone and build ealib

```bash
git clone https://github.com/kgskocelas/ealib-modern.git ealib
cd ealib
cmake .
make
```

CMake will find the Homebrew-installed Boost automatically.  `make` builds all targets.

### Out-of-tree builds (optional)

By default, `cmake .` and `make` place all their output files (compiled objects, libraries, and executables) directly inside the source tree alongside the source code.  An "out-of-tree" build keeps those generated files in a separate folder so your source directory stays clean.  To do this, replace the two build commands from Step 2 with:

```bash
cmake -B build
make -C build
```

`cmake -B build` tells CMake to create a folder called `build/` and put all of its output there instead of in the current folder.  `make -C build` tells Make to run inside that `build/` folder.  The finished executables will be in `build/examples/` instead of `examples/`.

---

### Step 3: Verify

Eight example executables are produced in the `examples/` directory:

| Executable         | What it exercises                                    |
|--------------------|------------------------------------------------------|
| `example-all-ones` | Simple all-ones genetic algorithm                    |
| `example-logic9`   | Avida-style logic-9 task                             |
| `example-lod`      | Line-of-descent EA                                   |
| `example-mp`       | Metapopulation EA                                    |
| `example-nsga2`    | Multi-objective NSGA-II                              |
| `example-qhfc`     | QHFC EA                                              |
| `example-pole`     | Pole balancing with neural networks (uses libann)    |
| `example-mkv`      | XOR task with Markov networks (uses libmkv)          |

Run the below installation verification test. It will create and populate a 'population_fitness.dat' file. When you open this file after running the test, the fitness should rise from roughly 0.40 toward 0.90 over 100 updates:

```bash
./examples/example-all-ones \
  --ea.representation.size 20 --ea.population.size 100 \
  --ea.generational_model.steady_state.lambda 10 \
  --ea.mutation.site.p 0.05 --ea.run.updates 100 \
  --ea.run.epochs 1 --ea.rng.seed 42 \
  --ea.statistics.recording.period 10
```

You can also try the other two examples that exercise the neural network and Markov network libraries.

**Pole balancing** (`example-pole`) — A simulated pole is balanced on a moving cart.  The evolutionary algorithm evolves a small neural network that learns to keep the pole upright.  Run with:

```bash
./examples/example-pole \
  --ea.representation.size 100 --ea.population.size 50 \
  --ea.mutation.site.p 0.05 --ea.mutation.normal_real.var 0.1 \
  --ea.generational_model.steady_state.lambda 5 \
  --ea.run.updates 30 --ea.run.epochs 1 --ea.rng.seed 42 \
  --ea.statistics.recording.period 30 \
  --ea.fitness_function.pole_balancing.max_steps 1000 \
  --ea.ann.input.n 5 --ea.ann.output.n 1 --ea.ann.hidden.n 5
```

> **Important:** `--ea.ann.input.n` must always be `5` for this example.  The pole-balancing fitness function sends exactly 5 numbers to the neural network (position, velocity, pole angle, pole angular velocity, and time).  If you change this to any other value the program will crash with an error.

When it finishes, it creates a file called `fitness.dat` in the current directory. The test passed if: the file exists, has two rows of numbers (plus the header), and the numbers look similar to the values below. With only 30 updates the improvement is small.

```
update mean_generation min_fitness mean_fitness max_fitness
0      0.0000          0.0060      0.0198       0.0750
30     1.9400          0.0200      0.0487       0.1530
```

---

**Markov network XOR** (`example-mkv`) — A population of Markov networks evolves to solve a logic puzzle (XOR).  Run with:

```bash
./examples/example-mkv -c etc/markov_network.cfg \
  --ea.run.updates 30 --ea.statistics.recording.period 30 --ea.rng.seed 42
```

This example also writes its output to a file called `fitness.dat`, overwriting the one from the pole balancing example above if you ran that first.  That is fine — they are separate tests. 

The test passed if: the file exists, has two rows of numbers plus the header (see example below), and the `max_fitness` column shows a value above 64 (meaning at least one network in the population is doing better than random guessing).  

```
update mean_generation min_fitness mean_fitness max_fitness
0      0.0000          50.0000     63.6800      77.0000
30     1.6400          40.0000     63.2800      77.0000
```

---

## Notes on Boost compatibility

EALib requires Boost ≥ 1.80 and C++14 or later.  The CMake build handles both automatically:

- `CMAKE_CXX_STANDARD` is set to **14** — required by `boost/math/tools/type_traits.hpp` in modern Boost (uses `std::decay_t`, `std::enable_if_t`, and similar C++14 type aliases)
- `BOOST_BIND_GLOBAL_PLACEHOLDERS` is defined globally — suppresses the Boost ≥ 1.73 deprecation warning for unqualified `_1`/`_2` placeholders used throughout libea headers
- `Boost::chrono` is linked explicitly alongside `Boost::timer` — prevents linker errors on platforms where `Boost::timer` does not carry the chrono dependency automatically

See `MODERNIZATION_NOTES.md` for the full technical record of these decisions.

---

## MSU HPCC

The section below was written based on current HPCC documentation (docs.icer.msu.edu).  The old 2019 HPCC instructions described building Boost 1.55/1.71 from source using b2/bjam — those no longer apply.  The HPCC now uses the Lmod module system and CMake is pre-installed as a module.  Whether a full Boost module is available can change as ICER updates their software stack; the instructions below cover both cases.

> **You need an HPCC account.**  If you do not have one, request access through ICER at https://icer.msu.edu before starting.

### Step 1: Connect and move to a development node

Open a terminal and log in to the HPCC:

```bash
ssh YOUR_NETID@hpcc.msu.edu
```

Replace `YOUR_NETID` with your MSU NetID.  You will be asked for your MSU password.

Move to a development node (Any `dev-*` node works):

```bash
ssh dev-intel18
```

### Step 2: Check whether a full Boost module is available

Run this command to search for Boost:

```bash
module spider Boost
```

- **If you see a result like `Boost/1.83.0-GCC-13.2.0`** (a version starting with just `Boost/`, not `Boost.Python/`), then a pre-built Boost is available.  Go to **Step 3a**.
- **If you only see results starting with `Boost.Python/`**, or nothing at all, then you need to build Boost yourself.  Go to **Step 3b**.

---

### Step 3a: Build using a pre-installed Boost module

Load a compatible set of GCC, CMake, and Boost.  The version numbers must match — use the versions you found in Step 2.  For example, if you saw `Boost/1.83.0-GCC-13.2.0`:

```bash
module purge
module load GCC/13.2.0
module load CMake/3.27.6-GCCcore-13.2.0
module load Boost/1.83.0-GCC-13.2.0
```

`module purge` clears any previously loaded modules to avoid conflicts.  Then skip ahead to **Step 4**.

---

### Step 3b: Build Boost from source (if no Boost module is available)

This takes about 5–10 minutes.  You only need to do it once.

First, load GCC and CMake:

```bash
module purge
module load GCC/12.3.0
module load CMake/3.26.3-GCCcore-12.3.0
```

Go to your home directory and download Boost 1.87.0:

```bash
cd $HOME
wget https://archives.boost.io/release/1.87.0/source/boost_1_87_0.tar.gz
tar -xzf boost_1_87_0.tar.gz
cd boost_1_87_0
```

Build and install Boost into a folder called `boost/` in your home directory:

```bash
./bootstrap.sh --with-toolset=gcc \
  --with-libraries=filesystem,iostreams,program_options,regex,serialization,system,timer,chrono \
  --prefix=$HOME/boost
./b2 install
```

`./bootstrap.sh` sets up Boost's own build tool.  `--with-libraries=...` selects only the compiled Boost libraries that EALib actually needs (skipping the rest saves time).  `--prefix=$HOME/boost` means the result goes into `~/boost/` rather than a system directory.  `./b2 install` compiles and copies everything into place.

When it finishes you should see a line like `...updated N targets...` with no errors.  You can now safely delete the source directory to free space:

```bash
cd $HOME
rm -rf boost_1_87_0 boost_1_87_0.tar.gz
```

---

### Step 4: Clone and build EALib

Go to your home directory and clone the repo:

```bash
cd $HOME
git clone https://github.com/kgskocelas/ealib-modern.git ealib
cd ealib
```

**If you followed Step 3a** (pre-installed Boost module):

```bash
cmake .
make
```

**If you followed Step 3b** (Boost built from source into `$HOME/boost`):

```bash
cmake -DBOOST_ROOT=$HOME/boost .
make
```

`-DBOOST_ROOT=$HOME/boost` tells CMake where to find Boost since it is not in a standard system location.  Everything else is the same.

`make` will take a minute or two.  When it finishes with no errors, the example executables are in the `examples/` directory.

### Step 5: Verify

Run the same verification test as the macOS instructions:

### Saving your module setup for future sessions

The modules you loaded with `module load` are only active during your current login session.  The next time you log in, they will be gone and the `cmake`/`make` commands will fail.  To avoid typing the same `module load` commands every time, save your current setup:

```bash
module save ealib-build
```

The next time you log in, restore it with:

```bash
module restore ealib-build
```
