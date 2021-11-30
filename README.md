# RIS-cluster-notes

# Frequently used commands

```bash
#build your docker image 
docker build -t .

#list all your docker images
docker images

#Run a docker image interactively
docker run -it 'image name'

#Change the name of a image
docker tag 'old name' 'new name'

# attache an existing container interactively
bsub -q general-interactive -Is -a 'docker_exec(job ID)' /bin/bash
```

# Creat your own image

```Dockerfile
# FILE: For l-picola and lcmetric.
# NOT for MPI run acrossing nodes

FROM ubuntu:18.04


# Install a bunch of build essentials.
RUN	apt-get update \
    && apt-get install -y \
        build-essential \
        cmake \
        git \
        sudo \
        unzip \
        emacs \
        wget \
                libhealpix-cxx-dev\
                python3\
                libfftw3-mpi-dev\
                libfftw3-dev\
                libopenmpi-dev\
                libgsl-dev\
                python3-pip;


RUN python3 -m pip install healpy
```

## On the cluster

First, make sure your home, storage1, and scratch1 directories are mounted to any docker image. Run

```bash
export SCRATCH1=/scratch1/fs1/jmertens
export STORAGE1=/storage1/fs1/jmertens/Active
```

after every time you log-in for interactive jobs, or add to your job script and ~/.bashrc file.

```bash
# check the storage quota at fs1, which is CACHE
mmlsquota --block-size auto -j jmertens_active cache1-fs1
# check the storage quota at scratch
mmlsquota -g compute-jmertens scratch1-fs1
# check the storage quota in the REAL storage place through SMB
smbclient -A .smb_creds -k //storage1.ris.wustl.edu/ris -U your username
```

Note that the */storage1/fs1/jmertens/Active* is just a Cache, its data will be moved to the storage space gradually (within 90 days). Data remains in the cache layer until the soft quota (2.5T) is reached, after that, data is moved to the storage (not 100% sure).

```bash
#Log in to docker hub
LSB_DOCKER_LOGIN_ONLY=1 bsub -G compute-jmertens -q general-interactive -Is -a 'docker_build' -- .

#asking for an interactive for test
bsub -Is -q general-interactive -a 'docker(python:slim)' /bin/bash

#use my own image
bsub -Is -q general-interactive -a 'docker(rectaflex/lpicola-lcmetric:latest)' /bin/bash

# attach on an existing job
bsub -q general-interactive -Is -a 'docker_exec(jobIShere)' /bin/bash 
```

## Submit a job script

A sample job script

```bash
#BSUB -n 64
#BSUB -q general
#BSUB -G compute-jmertens
#BSUB -J my_name
#BSUB -M 8000000
#BSUB -W 5
#BSUB -N
#BSUB -u chit@wustl.edu
#BSUB -o test.out
#BSUB -R 'rusage[mem=8GB] span[ptile=8]'
#BSUB -a 'docker(rectaflex/lpicola-lcmetric:latest)'

# This script utilizes 8 cpus on each node (by setting ptile=8), with a total of 64 cpus on 8 nodes. The [mem=8GB] reserves the memory of 8 GB for each node, so the total memory limit is 8*8 = 64 GB for all 8 nodes. 

cd /storage1/fs1/jmertens/Active/chit/lc_metric/lpicola/
make

other commands...
```

`bsub < your_script` to run it!

This script is pulling my own image `rectaflex/lpicola-lcmetric:latest`, you

Note that your */storage1* and */scratch1* will still be mounted after loading docker image after your set-up your *~/.bashrc* file correctly. So you only need to to compile and run!


# GPUS
List all available GPUs via `bhosts -gpu -w general`

ask for gpu
```
bsub ... -gpu "num=1:gmodel=TeslaV100_SXM2_32GB" ...
```

# Configure with open-mpi

Note the `openmpi` needs to be compiled with the LSF support so that the `mpirun` command will know the hosts configuration from the LSF system. This is actually very subtle task but fortunately, there is a tutorial 
(https://community.ibm.com/community/user/integration/blogs/john-welch/2018/11/26/compiling-open-mpi-with-ibm-spectrum-lsf-in-a-dock). 

Note that since the openmpi is compiled from sources, the software depending on the `openmpi` such as `fftw-mpi` needs also compiled from sources.

A sample `Dockerfile` supporting running L-picola can be found in 
https://github.com/ctian282/universal_dockerfile/blob/master/lsf_openmpi/Dockerfile


When running on the RIS cluster, adding envronmental variables in the `.bash_rc` (not enough by just adding it to the script)

```
export LSF_DOCKER_NETWORK=host
export LSF_DOCKER_IPC=host
```

`mpirun -np xxx --mca btl_vader_single_copy_mechanism none -mca btl_tcp_if_include ib0 'your command'`

# Usuful pages:

https://docs.ris.wustl.edu/doc/compute/recipes/ris-compute-storage-volumes.html
https://docs.ris.wustl.edu/doc/storage/03_storage.html#storage-limitations-calculating-free-space
https://docs.ris.wustl.edu/doc/compute/recipes/port-forwarding-gui.html#port-forwarding-gui

https://jira.wustl.edu/servicedesk/customer/kb/view/149551254?applicationId=05c4248c-e85c-3805-a287-01494ec5870a&portalId=2&pageNumber=1&resultNumber=1&q=linux&q_time=1615223626372

[https://www.bsc.es/support/LSF/9.1.2/lsf\_admin/index.htm?resources\_host_view.html~main](https://www.bsc.es/support/LSF/9.1.2/lsf_admin/index.htm?resources_host_view.html~main)
