# SLURM Setup [via SSH]
<img src="./figures/slurm_stdout_3.png" alt="slurm_stdout" align="right" width="600px" style="top:20px">

`jaynes` supports two SLURM launch modes. In the first mode, `jaynes` ssh tunnel through each time when you call `jaynes.run(train_fn)`. In the second model, we setup a `jaynes.server` instance on the SLURM cluster's login node, which we can control via TCP/IP callls. 

Depending on your cluster's admin policy and access configuration, one of these two modes would be more suitable. HPC admins can decide to offer a managed `jaynes.server`  endpoint where they provide token-based access.

In this tutorial, we will provide guide on the [ssh] launch model. For the client/server launch model, refer to [../05_client_server](../05_client_server) instead.

# Using Python on the Cluster

The shared network drive fabric might seem magic, it is not. In fact NFS fileIO can become a major bottleneck for an HPC network. This is why the recomended way to use python is to load python modules that the admin has setup. One one of the clusters for example, custom conda environments makes `import torch` take 87 seconds. Using system modules is faster because the admins can cache those files locally within each worker instance.

Basical module commands are:

- to see all available modules: `module avail`
- to load a module: `module load <module name>`
- to list all of the module you have loaded `module list`
- and to remove all modules: `module purge`

With jaynes, we can put this type of setup commands in the `runner.setup` field.

```bash
# to make the `module` command available
# source /etc/profile.d/modules.sh
module load cuda/10.2
module load anaconda/2021b
module load mpi/openmpi-4.0
```

## Getting Started

This example assumes that you can access a remote SLURM cluster via ssh. **First, make sure** ssh works:

```bash
ssh $JYNS_USERNAME@$JYNS_SLURM_HOST -i $JYNS_SLURM_PEM
```

to do so, you need to configure these in your environment script `./bashrc`

```bash
export JYNS_SLURM_HOST=/*your slurm cluster login node*/
export JYNS_USERNAME=/*your username*/
export JYNS_SLURM_PEM=~/.ssh//*your rsa key*/
export JYNS_SLURM_DIR=/home/gridsan/geyang/jaynes-mount
```

> password login are usually disabled on managed SLURM clusters.

**Second, install `jaynes`.** This tutorial is written w.r.t version: [0.7.2](https://github.com/geyang/jaynes/releases/tag/0.7.2).

```bash
pip install jaynes
```

## Managing Python Environment on A SLURM Cluster

Docker are usually not supported on managed HPC clusters, where a shared, NFS offer direct access to common python environments across machines. We assume that you have installed conda, and there is a `base` environment available.

First, let's installl `jaynes`, this is because jaynes use the `jaynes.entry` module to bootstrap the launch after the job starts. In most cases, you want to install all packages in your user space, which is why we pass in the `--user` flag.

```bash
conda activate base
pip install --user jaynes
```

The launch `.jaynes.yml` file contains the following values for configuring the remote python runtime. You can tweak this until your code runs.

```yaml
- !runners.Slurm &slurm
  envs: >-
    LC_CTYPE=en_US.UTF-8 LANG=en_US.UTF-8 LANGUAGE=en_US
  startup: >-
    source /etc/profile.d/modules.sh
    source $HOME/.bashrc
    module load cuda/10.2
    module load anaconda/2021a
    module load mpi/openmpi-4.0
```

This folder is structured as:

```bash
04_slurm_configuration
├── README.md
├── figures
└── launch_entry.py
```

Where the main file contains the following

### Main Script

```python
import jaynes

def train_fn(seed=100):
    from time import sleep

    print(f'[seed: {seed}] See real-time pipe-back from the server:')
    for i in range(3):
        print(f"[seed: {seed}] step: {i}")
        sleep(0.1)

    print(f'[seed: {seed}] Finished!')

if __name__ == "__main__":
    jaynes.config(verbose=False)
    jaynes.run(train_fn)

    jaynes.listen(200)
```

when you run this script, it should print out

<p><img src="./figures/slurm_stdout_single.png" alt="slurm_stdout_single" width="900px" style="top:20px"></p>

## Interactive vs Batch Jobs

When the `interactive` flag is `true`, we launch via the `srun` command. Otherwise, we pipe the bash script into a file, and launch using `sbatch`.

## Launching Multiple Jobs (Interactive)

Just call `jaynes.run` with multiple copies of the function:

```python
#! ./multi_launch.py
import jaynes
from launch_entry import train_fn

if __name__ == "__main__":
    jaynes.config(verbose=False)

    for i in range(3):
        jaynes.run(train_fn, seed=i * 100)

    jaynes.listen(200)
```

And the output should be 3 streams of stdout pipe-back combined together, running in parallel.

```bash
/Users/ge/opt/anaconda3/envs/plan2vec/bin/python /Users/ge/mit/jaynes-starter-kit/04_slurm_configuration/launch_entry.py
Jaynes pipe-back is now listening...
Running on login-node
Running inside worker
Running on login-node
Running on login-node
Running on login-node
Running inside worker
[seed: 200] See real-time pipe-back from the server:
[seed: 200] step: 0
[seed: 200] step: 1
Running inside worker
[seed: 200] step: 2
Running inside worker
[seed: 200] Finished!
[seed: 300] See real-time pipe-back from the server:
[seed: 300] step: 0
[seed: 300] step: 1
[seed: 300] step: 2
[seed: 100] See real-time pipe-back from the server:
[seed: 100] step: 0
[seed: 100] step: 1
[seed: 100] step: 2
[seed: 300] Finished!
[seed: 100] Finished!
[seed: 0] See real-time pipe-back from the server:
[seed: 0] step: 0
[seed: 0] step: 1
[seed: 0] step: 2
[seed: 0] Finished!
```

## Launching Sequential Jobs with `SBatch`
To submit a sequence of jobs with sbatch, 

1. turn off the `interactive` mode by setting it to `false`. 
2. specify `n_seq_jobs` to be > 1 (default: `null`).
3. make sure you set a job name, because otherwise, all of your `sbatch` calls will be sequentially ordered.

For example, `.jaynes.yml` may look like:

```yaml
- !runners.Slurm &slurm
  envs: >-
    LC_CTYPE=en_US.UTF-8 LANG=en_US.UTF-8 LANGUAGE=en_US
  startup: >-
    source /etc/profile.d/modules.sh
    source $HOME/.bashrc
  interactive: false
  n_seq_jobs: 3
```

Then, just call `jaynes.run(train_fn)` once:
```python
#! ./seq_jobs_launch.py
import jaynes
from launch_entry import train_fn

if __name__ == "__main__":
    for index in range(10):
        jaynes.config(verbose=False, launch=dict(job_name=f"unique-job-{index}"))
        jaynes.run(train_fn)
```
This runs `sbatch --job-name unique-job-0 -d singleton ...` for `n_seq_jobs=3` times, which requests sequential jobs.


## Issues and Questions?

Please report issues or error messages in the issues page of the main `jaynes` repo: [jaynes/issues](https://github.com/geyang/jaynes/issues). 

Happy Researching!  :heart:
