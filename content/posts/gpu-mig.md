+++
title = 'Multi-Instance GPU (MIG)'
date = 2024-09-17T10:33:17+05:30
draft = false
+++

Multi-Instance GPU is a way to securely partition/share GPU for CUDA applications, providing multiple users with separate GPU resources for optimal GPU utilization.

![Mig](/gpu-mig/gpu-mig-overview.jpg)

MIG allows us to partition single GPU card into a maximum of 7 GPU instance.

Below is the example of A100 40GB which is sharded into 8 memory slices and 7 compute slices.
Each of the memory slice will contain 1/8th of the total vram and each of the compute slice will have 1/7th total amount of streaming multiprocessors i.e. 8x5GB memory slice and 7 compute slices.

![Mig-a100](/gpu-mig/mig-a100.png)

### GPU Instance

To create a GPU partition/GPU Instance (GI) requires combining some number of memory slices with some number of compute slices. In the diagram below, a 5GB memory slice is combined with 1 compute slice to create a 1g.5gb GI profile.

![Mig-1g.5gb](/gpu-mig/1g.5gb.png)

#### Profile Placement

The number of slices that a GI can be created with is not arbitrary. The NVIDIA driver APIs provide a number of “GPU Instance Profiles” and users can create GIs by specifying one of these profiles.

![mig-profile-placement](/gpu-mig/mig-profile-placement.png)

For more details on supported MIG profiles check the Nvidia documentation - https://docs.nvidia.com/datacenter/tesla/mig-user-guide/#supported-mig-profiles
