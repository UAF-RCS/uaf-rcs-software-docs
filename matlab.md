# MATLAB



## MATLAB Usage

**Storage quotas and rates take effect July 1, 2018.** PIs and projects over quota in $CENTER1 and $ARCHIVE need to reduce data volumes or contact RCS immediately. Inaction will result in project member inability to read or write data starting July 1, 2018.

## Policy

Due to the shared nature of the Chinook login nodes RCS requests that MATLAB be run on the RCS Linux workstations for data processing and on the Chinook compute nodes for computationally intensive tasks. Please see the [Remote Login page](http://gi.alaska.edu/research-computing-systems/remote-login) for more information on running MATLAB on the workstations and the [MATLAB parallel computing section](https://www.gi.alaska.edu/research-computing-systems/matlab-usage#matlab-hpc) for information on running MATLAB on the compute nodes.

MATLAB sessions that are launched on the Chinook login nodes may be killed at RCS discretion due to these jobs affecting all users on the login nodes.

## MATLAB Parallel Computing

MATLAB jobs can execute tasks in parallel either using 24 cores on a single node, which requires no additional setup, or multiple cores on multiple nodes which may require some settings to be configured. Jobs on a single node can be submitted through Slurm, while jobs that use multiple nodes must be submitted through a MATLAB session.

### Parallel Jobs

Parallel jobs in MATLAB run on "parallel pools" which is a collection of MATLAB workers that will run tasks on the Chinook cluster. Some useful commands and things to keep in mind are the following:

* MATLAB workers do not have access to any graphical capabilities so data cannot be plotted in parallel
* Parallel jobs make use of the the parfor, parfeval, and/or the spmd MATLAB directives
* [parfor](https://www.mathworks.com/help/distcomp/parallel-for-loops-parfor.html) is used for loops that are not interdependent, and each iteration of the loop will be run on a separate core
* [pareval](https://www.mathworks.com/help/distcomp/asynchronous-parallel-programming.html) is used for functions that can be run asynchronously
* [spmd](https://www.mathworks.com/help/distcomp/execute-simultaneously-on-multiple-data-sets.html) stands for Single Program Multiple Data, and is for functions that operate on different data, but use the same code. This can spread each instance to its own worker, doing work in parallel. This is most often accomplished with the use of the labindex, the ID of an individual worker

For more information please see the [MATLAB Parallel Computing documentation](http://www.mathworks.com/help/distcomp/index.html)

### Single Node

Jobs that use only a single node \(up to 23 workers or threads in the debug or small queue\) may be submitted directly to the Slurm batch scheduler. When submitting a job directly to Slurm the parpool profile must be set to 'local' to use the CPUs available to the compute node. **Do not use the 'local' profile when submitting jobs through MATLAB or on the login nodes or Workstations as this will use as many cores as specified and will affect all users on a system**.

Submitting a MATLAB script to be run on compute nodes requires using the [Batch Submission System](http://www.gi.alaska.edu/research-computing-systems/hpc/chinook/using-batch-system). Your MATLAB script must initially set up a `parpool` which will launch the set of threads to be used for a parallel process. For example the following MATLAB script will create a parpool to generate a set of random numbers:

```text
%============================================================================
% parallelTest.m - Parallel Random Number - A Trivial Example
%============================================================================
% this will read in the "ntasks" provided by the Slurm batch submission script
parpool('local', str2num(getenv('SLURM_NTASKS')))
size = 1e7;
endMatrix = [];
tic % Start timing
parfor i = 1:size
    % Work that is independent of each task in the loop goes here
    endMatrix = [endMatrix; rand]
end
timing = toc; % end timing
fprintf('Timing: %8.72f seconds.n', timing);
delete(gcp); % Clean up the parallel pool of tasks
exit;
```

To submit this to the Chinook compute nodes a batch script must be created:  


```text
#!/bin/bash
#SBATCH --partition=$PARTITION
#SBATCH --ntasks=24
#SBATCH --tasks-per-node=24
#SBATCH --time=DD:HH:MM:SS
#SBATCH --job-name=$JOBNAME
module purge
module load slurm
module load matlab/R2016b
cd $SLURM_SUBMIT_DIR
matlab -nosplash -nodesktop -r "/path/to/parallelTest"
```

If this file were named parallelMatlab.slurm you would then submit it with:  
`chinook00 % sbatch parallelMatlab.slurm`

### Multiple Nodes

MATLAB jobs that use multiple nodes must be run in an interactive session. Some settings may need to be set by a user to allow MATLAB to interactively submit jobs to the Slurm batch scheduler.

Add the Slurm folder to MATLAB's PATH

**MATLAB GUI**

* Click the Set Path button in the toolbar
* Check if `/import/usrlocal/pkg/matlab/matlab-R2016b/toolbox/local/slurm`is in your path
* If not
  * Click Add Folder
  * Type in or copy `/usr/local/pkg/matlab/matlab-R2016b/toolbox/local/slurm` to the Folder name field
  * Click Save, save pathdef.m to `$HOME/.matlab/R2016b/pathdef.m`

**MATLAB Command Line**

* Run the `path` function
* Check to see if `/import/usrlocal/pkg/matlab/matlab-R2016b/toolbox/local/slurm` is in your path
* If not run `addpath('/usr/local/pkg/matlab/matlab-R2016b/toolbox/local/slurm');`

**Import the MATLAB Parallel Profile**

To use the MATLAB DCS you will need to import the Parallel Profile for Chinook. This can be done through the MATLAB GUI or the command line.

**MATLAB GUI**

* Click the Parallel drop down menu in the toolbar
* Click Manage Cluster Profiles
* Click Import
* Navigate to /usr/local/pkg/matlab/slurm\_scripts/
* Select chinook.settings and click Open

You should now have debug, t1small, t1standard, t2small, and t2standard in your Cluster Profile Window

**Command Line**

```text
>> clusterProfile = parallel.importProfile('/usr/local/pkg/matlab/slurm_scripts/chinook');
>> clusterProfile
```

You should see a 1x5 cell array containing 'debug','t1small','t1standard','t2small', and 't2standard'.

NumWorkers and NumThreads

Two other variables need to be set before running a job on the MATLAB DCS: NumWorkers and NumThreads.

NumWorkers is the number of MATLAB Workers available to the job. By default each Worker is assigned to a core on a node. Our license allows 64 Workers to be used across all users. You will need one more Worker than the number that your job requires because one Worker is dedicated to managing the others. For a two node job you can use up to 47 Workers for computations, and for three node jobs, 63 Workers.

Multiple threads can be assigned to a worker, if a Worker can benefit from multiple threads. NumWorkers \* NumThreads should be less than the total number of cores that are available. If not the job will use every core and may have unexpected behavior. There are two ways to set NumWorkers and NumThreads:

MATLAB GUI

* Click on the Parallel button in the toolbar
* Click on Manage Cluster Profiles
* Click on the Profile you wish to edit
* Click on Edit in the bottom right corner of the window
* In Number of workers available to cluster NumWorkers enter the number of workers \(must be less than 64**\)**
* In Number of computation threads to use on each worker NumThreads enter the number of threads

Command Line

```text
myCluster = parcluster('$PARTITION');
myCluster.NumWorkers = $NumWorkers; %less than 64
myCluster.NumThreads = $NumThreads
```

Running Jobs

Parallel jobs can be run during an interactive session of MATLAB by first setting up a Parallel pool on a partition. The Parallel pool will only start if there are nodes available. Starting a parallel pool can be started with the following commands in MATLAB:  
`myCluster = parcluster('$PROFILE');`  
where $PROFILE is the name of one of the queues on Chinook \(debug, t1/t2standard, t1/t2small\). This will start a parallel pool where can be done. Scripts or functions that are run in the interactive session of MATLAB that use parfor, parfeval, or spmd will run that code in the parallel pool that you have open.

When you are done with the parallel pool you will have to close it using the following command in MATLAB:  
`delete(gcp);`

Submitting Jobs

You can also submit jobs to the queue using MATLAB. The profiles that can be submitted to in this way are debug, t1small, t1standard, t2small, and t2standard, where each profile corresponds to a partition available on Chinook. For more information about the partitions on Chinook please see [our partition overview](http://gi.alaska.edu/research-computing-systems/hpc/chinook/using-batch-system#available-partitions).

To submit the job you will need to call the [MATLAB batch function](https://www.mathworks.com/help/distcomp/batch.html). The example below shows how to submit a function to be run on the compute nodes:

```text
%Leaving this option blank will use the chosen default profile%
myCluster = parcluster('$PROFILE');

%Run the script and pass its arguments. 'pool' sets the number workers%
%numWorkers must be at most 1 less than the total number of workers allocated in the profile%
%because MATLAB uses one additional worker to manage all the others%
myJob = myCluster.batch(@$FUNCTION,NumberOfOutputs,{input1,input2....,inputN},'pool',$NUMWORKERS);

%Wait for the job to finish before doing anything else%
wait(myJob);

%If submitted interactively do work on the output matrices%
WORK HERE

%Clean up the job%
destroy(myJob);
delete(gcp); 
```

where $PROFILE is the partition you wish to use, $FUNCTION is the function, in a .m file, that you wish to run, and $NUMWORKERS is the number of Workers you wish to use.

Customizing sbatch Parameters for MATLAB Job Submission

For multi-node jobs you may have to modify the CommunicatingSubmitFunction may need to be modified to shorten the Walltime on a job. The steps to do this are the following:

* Create a location for your custom MATLAB CommunicatingSubmitFunction `mkdir ~/matlab_scripts` for example
* Copy communicatingSubmitFcn.m from `/usr/local/pkg/matlab/slurm_scripts/communicatingSubmitFcn.m ~/matlab_scripts/customCommunicatingSubmitFcn.m`

  For clarity name your personal copy of communicatingSubmitFcn.m to something other than communicatingSubmitFcn.m to insure your profile calls the correct function.

* Modify the name of the function in customCommunicatingSubmitFcn.m as well
* Modify the following line in your customCommunicatingSubmitFcn.m:  


  ```text
  additionalSubmitArgs = sprintf('--partition=t1standard --ntasks=%d', props.NumberOfTasks);
  ```

  and add in `--time=D-HH:MM:SS` to the section in sprintf like so:

  ```text
  additionalSubmitArgs = sprintf('--partition=t1standard --time=1-12:00:00 --ntasks=%d', props.NumberOfTasks);
  ```

* After you've made your parcluster run the following command: `set(myCluster, 'CommunicatingSubmitFcn', @customCommunicatingSubmitFcn);` and run your job as normal

