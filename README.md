# HPC-net-bench

## Snellius GPU communication measurements

I compared A100 and H100 communication workloads across native and containerized runs. The [NVML sampler](GPU/include/nvmlClass.h) and [DCGM logger](GPU/include/dcgmiLogger.h) move power sampling off the benchmark timing path.

![NVLink versus PCIe energy comparison](https://0chonko.github.io/assets/blog/green-gpu-comms/energy-nvlink-vs-pcie.svg)

The [profiler write-up](https://0chonko.github.io/blog/measuring-without-mutating-designing-an-asynchronous-overhead-free-gpu-power-profiler/) explains the measurement setup. To repeat the hybrid H100 point-to-point experiment on a compatible Slurm system, follow the GPU setup below, then run `./sbatch/snellius-h100-hybrid/run-snellius-pp-all.sh` from `GPU/`. The scripts use Snellius-specific container and scheduler configuration.

## Results from the Snellius study

These benchmarks support German Savchenko’s 2024 Vrije Universiteit Amsterdam master’s thesis, *Unveiling the Performance Landscape of Modern HPC: Containerization, Network Performance, and Energy Footprint of GPU Interconnects*.

- [CPU networking](CPU/README.md#snellius-cpu-experiments) covers native, hybrid Apptainer, and fully contained runs on AMD EPYC Rome and Genoa. Hybrid bandwidth generally tracked native performance at larger message sizes.
- [GPU communication and power](GPU/README.md#snellius-gpu-experiments) covers native and hybrid runs on NVIDIA A100 and H100. The impact of containerization varied by operation, library, and node layout.

## Upstream benchmarks

The repository utilizes the cloud_noise and the interconnect benchmarks:

https://github.com/DanieleDeSensi/cloud_noise
https://github.com/HicrestLaboratory/interconnect-benchmark

---
#### Hardware Requirements:
- AMD Rome or Genoa CPUs, NVIDIA A100 or H100 GPUs.
- Infiniband/NVLinks for inter/intra-node communication.

#### Software Requirements:
- RHEL8, OpenMPI 4.1.5, CUDA 12.1, Driver version 545.
- Apptainer or Singularity for containerized environments.
- DCGM for power profiling.

---

## CPU Benchmarking

- Follow instructions from [de_sensi_noise_2022] to build the project

- Build the container with apptainer locally (with sudo) or on the cbuild partition (with --fakeroot). Transfer the completed images into the `images` directory if built locally.
    ```
    cd HPC-net-bench/CPU
    mkdir images
    apptainer build --fakeroot ~/hybrid_image.sif definitions/definition_hybrid.def
    apptainer build --fakeroot ~/contained_image.sif definitions/definition_contained.def
    mv ~/hybrid_image.sif images
    mv ~/contained_image.sif images
    ```
    For local build make sure to have the copies of definition files and run the following:
    ```
    sudo apptainer build hybrid_image.sif definition_hybrid.def
    sudo apptainer build contained_image.sif definition_contained.def
    ```

- Run scripts from `scripts-native`, `scripts-hybrid`, and `scripts-spack` to gather data with sbatch



---
## GPU Benchmarking

- Build GPU benchmarks using the following:
    ```bash
    make -f Makefile.SNELLIUS-POWER
    ```
- Build the container with apptainer locally (with sudo) or on the cbuild partition (with --fakeroot). Transfer the image to the repository on the cluster if built locally.
    ```
    cd HPC-net-bench/GPU
    mkdir containers/images
    apptainer build --fakeroot ~/hybrid_image.sif containers/definitions/definition_hybrid.def
    apptainer build --fakeroot ~/contained_image.sif containers/definitions/definition_contained.def
    mv ~/hybrid_image.sif containers/images
    mv ~/contained_image.sif containers/images
    ```

- Run experiments from the `sbatch` directory:
    ```bash
    ./sbatch/snellius-h100-hybrid/run-snellius-pp-all.sh
    ```
    Modify the `*all.sh` files to suit single-node or multi-node runs (halfnode.sh or wholenode.sh).

- The profiler files are located in ```include``` directory under *nvmlClass* and *dcgmiLogger*. Here the profiled fields can be modified and re-compiled. The ```-p 1``` flag can be used to collect NVML metrics, while ```-p 2``` is used to run DCGM profiler metrics. If the flag is ommited, no profiler is initiated. NOTE: the global $PROFILER_CHOICE value can be modified in `exports/vars.sh`. 


---

### Plotting Results

#### CPU:
- Use `plot.py` to generate latency and bandwidth plots by enabling/disabling specific methods at the bottom of the file.

#### GPU:
- Use `plotPowerDraw` for NVML power metrics, `plotPowerDrawDcgm` for DCGM power metrics, `plotPowerDrawDifferences` for native-hybrid percentile comparisons, and `plotBandwidthMultiPath.py` for bandwidth comparisons between native and hybrid runs.
- All of the plots take different directories to compare native and container data. Every file has a field with the two directories to consider. For instance, if we want to plot native vs hybrid on H100, go to the respective file and add the paths:
    ```python
    filename1 = "sout/native-h100"
    filename2 = "sout/hybrid-h100"
    ```

