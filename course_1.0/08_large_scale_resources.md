# PEARL Workshop

## Accessing PEARL

### On Windows
 - Download [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html).
 - Follow [these instructions](https://devops.ionos.com/tutorials/use-ssh-keys-with-putty-on-windows/#install-putty-and-puttygen) to setup PuTTY. Specifically the parts:
   - Install PuTTY And PuTTYgen
   - Use Existing Public And Private Keys
  
  
### On Linux

Assuming your username is `tmp100` and your key file is called `key_file.txt` you can use the following command to ssh into the PEARL system.

```bash
ssh -i tmp100 tmp100@ui.pearl.scd.stfc.ac.uk
```

## Singularity

### Pulling Images
There are lots of places where you can get existing container images for Singularity.

 - [Singularity Container Library](https://cloud.sylabs.io/library)
 - [Docker hub](https://hub.docker.com/search?q=&type=image)
 - [NVIDIA GPU Cloud](https://ngc.nvidia.com/catalog/all)
 
Some examples of pulling images from each of these locations. We've already downloaded the images we need, so don't run these during the session as they can take a long time to download:

```bash
singularity pull library://library/default/centos

singularity pull docker://tensorflow/tensorflow:latest-gpu

singularity pull docker://nvcr.io/nvidia/tensorflow:20.07-tf1-py3
```

### Running Containers

There are three different commands that can be used to run singularity containers:

 - `exec`: execute an arbitrary command inside the container. In the example below we run python inside the container, import tensorflow, and print the tensorflow version.

```bash
singularity exec tensorflow_latest-gpu.sif python -c "import tensorflow as tf; print(tf.__version__)"
```
 - `shell`: run the container and start an interactive shell prompt.
 
```bash
singularity shell tensorflow_latest-gpu.sif
```

 - `run`: the command specified by the image's `%runscript`. More on this in the next section.
```bash
singularity run tensorflow_latest-gpu.sif
```

### Building Containers
There are many, many prepackaged images out there, but what do we do if we want to make our own image? We can use Singularity build scripts to specify how to create an new image. For information on build scripts can be found in the [Singularity docs](https://sylabs.io/guides/3.6/user-guide/quick_start.html#build-images-from-scratch).

In this simple example we create a very minimal image that has a couple of additional applications packaged into it.

 - `Boostrap:`: where to get the base image from. Common options are:
   - `library`: the singularity container repos
   - `docker`: a docker image from docker hub or elsewhere
   - `localimage`: a singularity image on your machine
 - `From:`: the name of the image to pull from the repository. In this case we're pulling a blank Ubuntu image from the singularity hub.
 - `%post`: this is the heart of the build script. This is where you write the commands to install additional packages ontop of the base image.
 - `%environment`: set any environment variables for the image.
 - `%runscript`: the command to run when the user starts the container with the `singularity run` command.

```
BootStrap: library
From: ubuntu:16.04

%post
    apt-get -y update
    apt-get -y install fortune cowsay lolcat

%environment
    export LC_ALL=C
    export PATH=/usr/games:$PATH

%runscript fortune | cowsay | lolcat
```

To build the image we use the `build` command. The first argument is the name of the output image file. The second argument is the buildscript definition file. The `--fakeroot` option is required to 

```bash
singularity build --fakeroot lolcow.sif lolcow.def
```

Finally we can run the image we created:

```bash
singularity run lolcow.sif 
```

## SLURM
SLURM is a job scheduling system. It can be used to schedule jobs on a shared resource such as PEARL. Jobs submitted to the queue will be run when resources (in our case available GPUs) become available.

We can interact with SLURM to find out the status of the SLURM job queues using the following commands:

```bash
#show all job queues
sinfo 
#show all pending or running jobs
squeue
#show previously run jobs for user
sacct
#show pending/running jobs for user pearl008
squeue –u pearl008
```

We can run an interactive shell with the following command. This will allocated a single GPU resource on the PEARL system and provide a bash prompt. Note that the `--reservation=Diamond2020` is only required during the workshop.

```bash
srun --gres=gpu:1 --reservation=Diamond2020 --pty bash 
```

An example of a very basic batch script. Each of the `#SBATCH` lines sets a different option in SLURM. Again, as above, the `--reservation=Diamond2020` is only required during the workshop. Each of the options are as follows:

 - `gres`: set the generic resources for this job. Here we're specifying we want one GPU.
 - `job-name`: set the display name of the job. This will be visible in the public queue.
 - `time`: set the estimated maximum time of the job. Jobs that run beyond this value will be kill automatically.
 - `mem`: set the required CPU RAM for the job. On the DGX system you have up to 1.5TB of shared RAM available. But you will only have 16GB on GPU RAM per GPU.

```bash
#!/bin/bash
#SBATCH --reservation=Diamond2020
#SBATCH --gres=gpu:1
#SBATCH --job-name ="ML Job"
#SBATCH --time=0-00:10:00
#SBATCH --mem=1GB

nvidia-smi
```
We can submit our job to the queue with the following command:

```bash
sbatch simple-job.job
```

You can run any arbitrary bash command in your job script. For example, we can run singularity containers. Remember if we want the GPU to be accessible we must add the `--nv` flag to the singularity command!

```bash
#!/bin/bash
#SBATCH --reservation=Diamond2020
#SBATCH --gres=gpu:1
singularity exec --nv pytorch_20.06-py3.sif nivida-smi
```

We can then submit this job with the following command:

```bash
sbatch simple-singularity-job.job
```

## Exercises

- Exercise 1:
  - There is a script called `train-fmnist.py`. This trains the same dense model from the first lesson on neural networks on the fashion MNIST dataset.
  - Write a batch script to run this with the `tensorflow_latest-gpu.sif` container.
  - Submit the job and check the progress

- Exercise 2:
  - Create a script for the autoencoder example from the autoencoder practical
  - Write a batch script to run this with the `tensorflow_latest-gpu.sif` container.
  - Submit the job and check it runs