
This repo uses ansible to configure a host to run an containerised openfoam benchmark using **single-node** MPI: https://develop.openfoam.com/committees/hpc#3-d-lid-driven-cavity-flow

# Key points

- Both the ansible and target host are assumed to be running `RockyLinux 8.x`.
- The OpenFoam install is containerised and the container includes its own (fairly-recent) MPI:

        77f99f65b012: ~>> mpirun --version
        mpirun (Open MPI) 4.0.3

- The "install" is actually a script http://dl.openfoam.org/docker/openfoam10-linux which pulls and launches the container.
- This launcher script mounts a host directory into the container: by default this is the pwd, but it can’t be the user’s homedir (presumably for security)
- The repo contains the following playbooks:
    - `docker.yml`: Installs docker
    - `install.yml`: Installs the launcher script and creates a suitable openfoam directory
    - `benchmark-3dcavity.yml`: Clones the benchmark into that openfoam directory, and fixes a couple of things

# Further documentation
The ansible and instructions here are derived from:
- https://develop.openfoam.com/committees/hpc#3-d-lid-driven-cavity-flow
- https://www.openfoam.com/documentation/tutorial-guide/2-incompressible-flow/2.1-lid-driven-cavity-flow
- https://www.openfoam.com/documentation/user-guide/3-running-applications/3.2-running-applications-in-parallel

# Ansible Host Setup


    dnf install -y python38
    python3.8 -m venv venv
    . venv/bin/activate
    pip install -U pip
    pip install ansible
    pip install python-openstackclient

# Configuring Target Host

1. Create an inventory e.g.:

        # ./inventory
        openfoam ansible_host=192.168.3.226

        [all:vars]
        ansible_user=rocky

2. Run:

        ansible-playbook -i <inventory_file> <playbook>


There are no options in playbooks except for `benchmark-3dcavity.yml` where you may want to set:
- `problem_size`: str `S`, (default) `M` or `L`.
- `parallelisation`: str `threads` (default), `cores` or `none`.

**Note**: I'm not sure `threads` will be fastest!

# Running the benchmark

1. ssh into the target host

2. Run:

        [rocky@sbtest-2 rocky-10]$ cd /home/rocky/OpenFOAM/rocky-10/hpc/Lid_driven_cavity-3d/<problem_size>/
        [rocky@sbtest-2 S]$ openfoam10-linux
        77f99f65b012: ~>> blockMesh
        77f99f65b012: ~>>decomposePar -force # -force required if paralelisation changed
        77f99f65b012: ~>> time mpirun --use-hwthread-cpus -np <num_proc> icoFoam -parallel

    where:
    - `<problem_size>` is as used for benchmark-3dcavity.yml (default `'S'`).
    - `<num_proc>` should be the number of threads for `parallelisation: 'threads'` (default) or number of cores for `parallelisation: 'cores'`.
    - `--use-hwthread-cpus` should be omitted for `parallelisation: 'cores'`

3. Wait for it to finish. It runs 0.5s of simulation.

# Timings

On a 4x Broadwell VCPU / 4GB RAM VM, for problem size `S`, wallclock times were:
- serial: 103m53s
- 4x threads: 60m40.243s (~1.7x speedup)
