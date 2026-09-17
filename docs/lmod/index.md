# Software modules (Lmod)
 
Software is made available through [Lmod](https://lmod.readthedocs.io) on all nodes of the cluster. Everything lives under `/opt/software`, which is installed on the login node and mounted over NFS on every other node. A module loaded on the login node therefore behaves the same way inside a Slurm job.

## Quick start
 
```bash
module avail                 # list available software
module load julia            # load the default version
module load julia/1.12.7     # load a specific version
module list                  # show what is loaded
module unload julia          # remove one module
module purge                 # remove all modules
```

## Conda (miniconda3)
 
```bash
module load miniconda3
conda create -n myenv python=3.12 numpy pandas
conda activate myenv
```
 
- Environments are created in `~/.conda/envs`. Home directories are shared, so an environment created on the login node also works inside jobs.
- Packages are downloaded to a shared cache in `/opt/software/conda/pkgs`. As this grows creating a new env is fast.
- Packages come from conda-forge only. The Anaconda `defaults` channel is disabled because it requires accepting Anaconda's terms of service.
- One shoud not run `conda init`. Loading the module sets up the `conda` command, and `conda init` would write a fixed path into your `~/.bashrc`.


## Layout
 
```
/opt/software/
├── lmod/
│   ├── lmod -> <version>        # symlink used by all init scripts
│   └── <version>/
├── modulefiles/                 # MODULEPATH, one directory per package
│   ├── julia/<version>.lua
│   └── miniconda3/<version>.lua
├── julia/<version>/
├── miniconda3/<version>/        # .condarc for all users lives here
└── conda/pkgs/                  # shared conda package cache, writable by everyone
```

 
## Node setup
 
Every node that uses modules needs:
 
- the NFS mount in `/etc/fstab`:
  `10.157.154.9:/opt/software  /opt/software  nfs4  _netdev,hard,noatime  0 0`
- the Lmod runtime packages: `lua5.3 lua-posix lua-bit32 tcl`
- `/etc/lmod/.modulespath` containing `/opt/software/modulefiles`
- the symlink `/etc/profile.d/z00_lmod.sh -> /opt/software/lmod/lmod/init/profile`
Nodes should run the same Ubuntu release as amygdala, where Lmod was built. The Ansible roles set all of this up.

## Adding software
 
Install the software into `/opt/software/<name>/<version>` and add a modulefile at `/opt/software/modulefiles/<name>/<version>.lua`. A minimal modulefile:
 
```lua
whatis("Name: Example")
whatis("Version: " .. myModuleVersion())
 
local root = pathJoin("/opt/software/example", myModuleVersion())
prepend_path("PATH", pathJoin(root, "bin"))
prepend_path("LD_LIBRARY_PATH", pathJoin(root, "lib"))
```
 
`myModuleVersion()` takes the version from the file name, so a new version only needs a copy of the file under the new name. Check the syntax before users load it:
 
```bash
luac5.3 -p /opt/software/modulefiles/example/1.0.lua && echo "syntax OK"
module show example/1.0
```
 
Paste modulefiles with care. A dropped character in a pasted heredoc (for example `elp(` instead of `help(`) causes a `syntax error near <eof>` when the module is loaded.
 
When several versions exist, Lmod loads the highest one by default. To pin a different default, create a symlink named `default` in the package directory:
 
```bash
ln -s 1.12.7.lua /opt/software/modulefiles/julia/default
```

## Upgrading Lmod
 
```bash
cd /usr/local/src/Lmod-<new-version>
./configure --prefix=/opt/software
make install
```
 
`make install` installs into `/opt/software/lmod/<new-version>` and moves the `lmod` symlink. Nodes pick up the new version at the next login. To test a version without switching users over, run `make pre-install` instead, which skips the symlink.