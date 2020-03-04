# Swarm

Swarm is a script designed to simplify submitting a group of commands to the Biowulf cluster.

This version has been forked from the original NIH one, to run at Pawsey Supercomputing Centre.


## Quick start

Suppose inyour work directory you have a text file called `list`, that contains a long list of serial commands to be executed through Slurm.

* To run them as one job/one core per command type: `swarm -f list`
* To run as job packs, to fill entire nodes of the cluster, run: `swarm -f list -p auto`

You can set memory and thread requirements using `-g` and `-t`, respectively.  Multi-threaded swarms are currently not compatible with multi-packed swarms.

Run `swarm -h` for additional options.


## How swarm works

The `swarm` script accepts a number of input parameters along with a file containing a list of commands that otherwise would be run on the command line.  `swarm` parses the commands from this file and writes them to a series of command scripts.  Then a single batch script is written to execute these command scripts in a slurm job array, using the user's inputs to guide how slurm allocates resources.

### Packing

Packing a swarm means running multiple commands per subjob in parallel, by allocating multiple cores and running one command per core.  This can be quite useful when running on systems (such as Magnus at Pawsey Supercomputing Centre) where the minimum job allocation is an entire node; packing allows to maximise usage of hardware resources.

### Bundling versus folding

Bundling a swarm means running two or more commands per subjob serially, uniformly.  For example, if there are 1000 commands in a swarm, and the bundle factor is 5, then each subjob will run 5 commands serially, resulting in 200 subjobs in the job array.

Folding a swarm means running commands serially in a subjob only if the number of subjobs exceed the maximum array size (`maxarraysize` = 1000).  This is a new concept.  Previously, if a swarm exceeded the `maxarraysize`, then either the swarm would fail, or the swarm would be autobundled until it fit within the `maxarraysize` number of subjobs.  Thus, if a swarm had 1001 commands, it would be autobundled with a bundle factor of 2 into 500 subjobs, with 2 commands serially each.  With folding, this would still result in 1000 subjobs, but one subjob would have 2 serial commands, while the rest have 1.


## Behind the scenes

`swarm` writes everything in your user-specific scratch directory:

```
/scratch/$PAWSEY_PROJECT/$USER/swarm_$PAWSEY_CLUSTER
├── 4506756 -> YMaPNXtqEF
└── YMaPNXtqEF
    ├── cmd.0
    ├── cmd.1
    ├── cmd.2
    ├── cmd.3
    └── swarm.batch
```

`swarm` (running as the user) first creates a subdirectory within the user's directory with a completely random name.  The command scripts are named `cmd.#`, with `#` being the command index within the job array.  The batch script is simply named `swarm.batch`.  All of these are written into the temporary subdirectory.

### Details about the batch script

The batch script `swarm.batch` hard-codes the path to the temporary subdirectory as the location of the command scripts.  This allows the swarm to be rerun, albeit with the same sbatch options.

The module function is initialized and modules are loaded in the batch script.  This limits the number of times `module load is called to once per swarm, but it also means that the user could overrule the environment within the swarm commands.

### What happens after submission

When a swarm job is successfully submitted to slurm, a jobid is obtained, and a symlink is created that points to the temporary directory.  This allows for simple identification of swarm array jobs running on the cluster.

If a submission fails, then no symlink will be created.

When a user runs swarm in development mode (`--devel`), no temporary directory or files are created.


## Testing

Swarm has several options for testing things.

`--devel:` This option prevents `swarm` from creating command or batch scripts, prevents it from actually submitting to sbatch, and prevents it from logging to the standard logfile.  It also increases the verbosity level of `swarm`.

`--verbose:` This option makes `swarm` more chatty, and accepts an integer from between 0 (silent) and 4.  Running a swarm with many commands at level 4 will give a lot of output, so beware.

`--debug:` This option is similar to `--devel`, except that the scripts are actually created.  The temporary directory for the `swarm.batch` and command scripts begins with `dev`, rather than `tmp` like normal.

`--no-run:` A hidden alacarte option, prevents `swarm` from actually submitting to sbatch.

`--no-log:` A hidden alacarte option, prevents `swarm` from logging.

`--logfile:` A hidden alacarte option, redirects the logfile from the standard logfile to one of your choice.

`--no-scripts:` Don't create command and batch scripts.


## Logging

* `swarm` logs to `$MYSCRATCH/swarm_$PAWSEY_CLUSTER/logs/swarm.log`
* `swarm_cleanup.pl` logs to `$MYSCRATCH/swarm_$PAWSEY_CLUSTER/logs/swarm_cleanup.log`


## Index File

An index file `$MYSCRATCH/swarm_$PAWSEY_CLUSTER/logs/swarm_tempdir.idx` is updated when a swarm is created.  This file contains the creation timestamp, user, unique tag, number of commands, and P value (either 1 or 2):

```
1509019983,mmouse,e4gLIFwqhq,1,1
1509020005,mmouse,aFwi3QYiQ0,13,1
1509020213,dduck2,jqcJTSiIBH,3,1
1509020215,dduck,qqBMb2SLzl,1,1
1509020225,ggoofy,64PZ3h80nB,1000,1
```
