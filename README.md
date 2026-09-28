## Requalizer -- Middleware '26 Artifact

[![DOI](https://zenodo.org/badge/latestdoi/123456789)](https://doi.org/10.5281/zenodo.12345678)

This repository contains the artifact for the Middleware '26 paper:

> Requalizer: A Co-designed Information Flow Control and Quality of Service Management Framework


### Overview

This repository contains the following items:

* `src/` - The source code of Requalizer.
* `data/` - The original experimental data used to produce Figure 8, Figure 9, Table 3, and Table 4 in the paper.
* `scripts/` - The scripts for reproducing the data and creating the figures and tables.
* `Dockerfile` - for creating the container with all the necessary dependencies needed for running the experiments.


### Platform Requirements

Requalizer was evaluated on a cluster of eight virtual machines, each with 2 vCPUs and 4 GB RAM, emulating a small edge-to-cloud infrastructure. 

We recommend that the machine has at least 10 GB of disk space available for the experiment data.

You can run the artifact on any machine using Docker, but the performance you observe might be slightly different than those in the paper due to platform differences.

---

### Getting Started

The quickest and recommended way to get started with running the artifact is to use the [pre-built Docker image](https://hub.docker.com/r/jungkumseok/requalizer) available at Docker Hub (`jungkumseok/requalizer:middleware26`), as it has the experiment environment already prepared with all the dependencies installed and workloads copied. The `Dockerfile` used to build the image can be found in this repository. 

Before starting a new container from the pre-built image, you must first decide whether you want to mount any volume. This artifact *does not* require any volume to be mounted, but if you want to easily access any data produced inside the container, we suggest you mount a directory from your host machine to the container path `/root/output`. The experimental scripts are configured to save all data to `/root/output`.

Start a new container and enter the interactive shell:

```
docker run -it --name requalizer-exp jungkumseok/requalizer:middleware26 /bin/bash
```

(OPTIONAL) Or, start a new container with a mounted volume (assuming the directory on the host machine to be mounted is `/home/user/requalizer-output`):
```
docker run -it --name requalizer-exp --mount type=bind,source=/home/user/requalizer-output,target=/root/output jungkumseok/requalizer:middleware26 /bin/bash
```


### Reproducing the Experiments

Assuming that we are now in the container environment, this section will walk through the steps for reproducing the results from the paper.

#### Producing the Figures and Tables using the Original Data

First, as a sanity check, let us simply run the scripts for creating the figures and tables, using the original data from the paper. Navigate to `/root/requalizer/scripts/presentation`:

```
# When the container starts, the CWD is /root
cd requalizer/scripts/presentation
```

The following are the scripts for producing Figure 8, Figure 9, Table 3, and Table 4:
```
plot-latency-histogram.py
plot-throughput-snapshot.py
generate-violations-table.py
generate-mttr-table.py
```

To run any of the scripts, we must activate the Python virtual environment. Activate `venv`:
```bash
# CWD: /root/requalizer/scripts/experiment
source ../presentation/.venv/bin/activate
```

Then, run the scripts as the following:
```
python plot-latency-histogram.py /root/requalizer/data/experiment-1/latency-data.csv
python plot-throughput-snapshot.py /root/requalizer/data/experiment-1/throughput-data.csv
python generate-violations-table.py /root/requalizer/data/experiment-1/violations-data.csv /root/requalizer/data/experiment-2/violations-data.csv
python generate-mttr-table.py /root/requalizer/data/experiment-2/mttr-data.csv
```

The above scripts should produce the figures and tables in the `/root/output` directory. The files will be named as below (`YYYYmmdd_HHMMSS` replaced with the appropriate timestamps):
```
latency-histogram.YYYYmmdd_HHMMSS.png
throughput-snapshot.YYYYmmdd_HHMMSS.png
violations-table.YYYYmmdd_HHMMSS.txt
mttr-table.YYYYmmdd_HHMMSS.txt
```

If you have mounted a host directory, you should be able to see these files in the host machine. If not, you will need to `docker cp` the files into the host machine.
```
# From the host machine
docker cp requalizer-exp:/root/output/latency-histogram.YYYYmmdd_HHMMSS.png /path/on/my/machine/latency-histogram.png
```

You can compare the figures and tables you produced with the original ones included in the paper to verify that the scripts ran successfully.


#### Examining the Core Algorithms (Reference Implementations)

Before running the full system experiments on the test cluster, you can examine Requalizer's core algorithms using the Python reference implementations in this repository. These scripts extract the core algorithmic logic out of the middleware (written in C#), allowing you to examine the contributions without needing to spin up a test cluster. 

The reference scripts are located in `/root/requalizer/scripts/experiment/`. To run them, activate the Python virtual environment (if you have not already):
```bash
# CWD: /root/requalizer/scripts/experiment
source ../presentation/.venv/bin/activate
```

**1. Workload Placement / Scheduling (`cp-scheduler.py`)**
This script demonstrates the dataflow-aware workload placement strategy (Section 4.3). It uses Google OR-Tools Constraint Programming to map predictive services to physical hosts, optimizing for bandwidth and latency while strictly adhering to DIFT isolation tags and group anti-affinity.
You can test the scheduling decisions for different applications (AAL, FD, SPG) in aware or unaware modes, or run a scalability test.
```bash
# Run the scheduler for the Ambient Assisted Living (AAL) topology
python cp-scheduler.py --app AAL --mode aware

# Or run the scalability test for a 16-node cluster with 10 parallel services
python cp-scheduler.py --app SCALE --hosts 16 --services 10
```

**Interpreting the Scheduler Output (AAL Example)**
If you run the AAL application in `aware` mode (`python cp-scheduler.py --app AAL --mode aware`), you can observe Requalizer's algorithms in action:
* **The Host Setup**: The script models an 8-node edge-to-cloud cluster. Hosts 0-2 are in the `public` zone, 3-5 are `internal` cloud servers, and 6-7 are edge devices located at the `patient`'s home (equipped with specific sensors like `camera` or `wearable`).
* **The Scheduling Problem**: We must place 11 microservices of the AAL pipeline onto these 8 hosts. The placement must respect resource limits (CPU/RAM), network topology (bandwidth/latency), and most importantly, the DIFT isolation rules (e.g., highly sensitive patient video data cannot be processed on a `public` node). Furthermore, the 4 instances of the `Notifier` service belong to a replica group and must be spread across different hosts to ensure resilience (anti-affinity).
* **The Solution**: The OR-Tools solver outputs an optimal placement mapping. You will observe that:
  * `VideoCamera` and `Wearable` are strictly pinned to Hosts 6 and 7 (the edge devices with the matching hardware and `patient` tags).
  * Sensitive processing components like `AIDoctor` and `RecordManager` are strictly isolated to the `internal` cloud nodes (Hosts 3, 4, or 5).
  * The `Notifier` replicas are successfully distributed across distinct hosts (solving the anti-affinity constraint for high availability) while individually respecting their distinct clearance levels (e.g., `Notifier 4`, which handles non-sensitive aggregate statistics, is allowed to run on a `public` Host like 0 or 1).

This correctly mirrors the behavior described in **Section 4.3 (Workload Placement)**, proving that the Constraint Programming (CP) formulation mathematically guarantees strict DIFT compliance and resilience prior to deployment.

**2. DIFT-Aware Load Balancing (`load-balancer-advanced.py`)**
This script implements the Mixed-Integer Linear Program (MILP) formulation of the DIFT-aware Load Balancer (Algorithm 1, Section 4.4). It calculates routing flow to guarantee minimum replica availability ($c$) for failure tolerance without violating DIFT rules.
```bash
# Calculate flow matrices ensuring a minimum redundancy of 2 active routes per label
python load-balancer-advanced.py --redundancy 2
```

**3. Dataflow & Correctness Simulator (`dift-simulator.py`)**
This discrete-event simulator validates Requalizer's routing mechanisms and label propagation (RQ2: Correctness). It evaluates different node architectures under dynamic conditions, allowing you to observe the exact routing decisions and dataflow behavior.
You can simulate any of the three applications (AAL, FD, SPG) across different dynamic conditions (`aware stable`, `unaware stable`, `unaware crash`, `unaware congestion`).
```bash
# Simulate 10,000 messages through the AAL topology with DIFT-aware routing
python dift-simulator.py --app AAL --mode 'aware stable' --messages 10000

# Simulate the Smart Power Grid (SPG) topology under network congestion without DIFT awareness
python dift-simulator.py --app SPG --mode 'unaware congestion' --messages 10000
```


#### Running Experiment 1: Efficiency and Correctness (Stable Conditions)

We will now run the first experiment described in Section 5.4 of the paper, which evaluates Requalizer under stable conditions (no machine or network failures).

Navigate to the `/root/requalizer/scripts/experiment-1` directory.
```
# CWD: /root/requalizer/scripts/presentation
cd ../experiment-1
```

Run the experiment script:
```
# CWD: /root/requalizer/scripts/experiment-1
./run-stable-experiments.sh
```

This script will run the three benchmark applications (Ambient Assisted Living, Fraud Detection, Smart Power Grid) under three system configurations (Baseline, Layered DIFT, and Requalizer).

Once the experiments have finished, there will be a directory in `/root/output` containing the raw measurements from each run.

Navigate back to the presentation scripts to process the results:
```
cd ../presentation
node compile-stable-results.js /root/output/exp1-YYYY-mm-dd
```
This will generate the CSV files `latency-data.csv`, `throughput-data.csv`, and `violations-data.csv`, which you can then pass to the Python plotting scripts described earlier to generate Figure 8, Figure 9, and the stable conditions portion of Table 3.


#### Running Experiment 2: Resilience (Dynamic Conditions)

Next, we run the second experiment described in Section 5.5, which evaluates Requalizer under node failures and network congestion.

Navigate to the `/root/requalizer/scripts/experiment-2` directory.
```
cd ../experiment-2
```

Run the experiment script:
```
# CWD: /root/requalizer/scripts/experiment-2
./run-dynamic-experiments.sh
```

This script will run the applications while injecting node failures (crashing critical components) and link congestion. It measures the Mean Time To Recover (MTTR) and monitors for DIFT violations during recovery and congestion.

Once finished, compile the results:
```
cd ../presentation
node compile-dynamic-results.js /root/output/exp2-YYYY-mm-dd
```
This will generate `mttr-data.csv` and the dynamic `violations-data.csv`. Use the Python scripts to generate Table 4 and the dynamic conditions portion of Table 3.


### (Optional) Building the Docker Image

In case you want to build the Docker image yourself, you can use the `Dockerfile`.
Assuming you have cloned this repository, simply run the following command in this repository's root:
```
docker build -t my-requalizer-image:1.0 .
```