[title]: - "Troubleshooting Job Issues"

[TOC]


# Learning Objectives

This guide discuses how to diagnose and resolve issues with jobs, including 
- Providing an overview of different ways to diganose job issues
- How to investigate jobs that remain on idle
- How to investigate and resolve issues with held jobs
- How to identify if jobs are encountering issues with specific OSPool sites

# Overview

Like all computing clusters that utilize a job submission system, jobs run by HTCondor on the OSPool can encounter different types of errors. A majority of errors are internal job errors unintentionally created by the user. For example, insufficient resource requests or incorrectly specified file names/paths will cause issues with job execution. In this guide, we will discuss how to identify errors such as these that may cause a job to sit on hold indefinitely or prevent it from successfully completing.

More rarely, jobs can also encounter issues that are external to the job and not always predictable in nature. The OSPool is composed of "heterogeneous" compute resources donated from around the world. The ability to incorporate different compute resources contributed by universities, national labs, and other research-supporting sites around the world into the OSPool is why this virtual pool so large and computationally powerful. The OSPool is continually expanding to incorporate new compute resources allowing billions of compute hours to be carried out each year. The dynamic nature of the OSPool means that in rare cases, a job may not be compatible with running at a specific site or on a certain type of machine. In this guide, we discuss how to identify if your job is having site-related issues and how to circumvent these errors. 


# Monitoring Job Status using condor_q

After you submit a job to HTCondor using `condor_submit`, it is possible to monitor the status of the job using `condor_q`. By default, `condor_q` will show only your jobs grouped into batches. To see the status of individual jobs, use `condor_q -nobatch`. Each job can be in one of four states: Idle, Run, Done, or Held. 

Idle jobs are waiting for the resources requested in the submit file to become available. Once matched to an available slot, jobs will begin running ("Run") and will either complete ("Done") or be placed on hold ("Held"). Held state means that the job has encountered some error, and cannot run. This doesn't necessarily mean that your job has failed, but, for whatever reason, Condor cannot fulfill your request(s). Once a job goes on hold, it will not complete without action from the user. **In most cases, it is not possible to obtain the any files created thus far in the job once it is held.**


# Job Remaining Idle and Not Starting

Sometimes jobs may take longer to start or appear to remain "Idle" indefinately. When this occurs, it is possible to investigate the reason that a job has not matched to a machine using `condor_q <JobID> -better-analyze` or `condor_q <JobID> -analyze`. These commands will list a variety of information including the number of machines currently available in the OSPool and the number of machines that your job can run on based off the information listed in the submit file. 

Imagine a scinero where we want a job to run at one of two research sites, but incorrectly specified that we want our job to run at *both* sites. An example of a requiremnts line that may request a job to run at both sites would be:

```Requirements = (GLIDEIN_Site == "SLATE_US_NMSU_DISCOVERY" && GLIDEIN_Site == "Crane")```

Because it is not possible to run a job on a machine found both at New Mexico State University (NMSU) and Crane University (Crane), this job will appear to remain idle indefinitely as it continues to search for a machine that matches these requirements. 

By using `-better-analyze` or `-analyze`, we can investigate the number of machines capable of running this job: 

```
21562138.000:  Run analysis summary ignoring user priority.  Of 3257 machines,
   3257 are rejected by your job's requirements
      0 reject your job because of their own requirements
      0 match and are already running your jobs
      0 match but are serving other users
      0 are able to run your job

WARNING:  Be advised:
   No machines matched the jobs's constraints
```
In the example above, there are 3,257 machines in the OSPool when job 21562138.000 was submitted. Despite thousands of machines being present in the pool, zero machines satisfy the requirements specified in the submit file. Since HTCondor cannot satisfy the job requirements specified by the user, the job will not start. 

## Example Idle Job Case Study: Jobs with Large Resource Requests

`-better-analyze` and `-analyze` can also provide insight as to why certain jobs with large resource requests or certain requirements specified may take longer to start. This is because jobs with few matches relative to the available number of machines will be expected to take longer to start. 

For example, if we submit a job with `request_memory = 2TB` in the submit file, we can use `condor_q -better-analyze Job-ID` to see why this job will not start: 

```
$ condor_q -better-analyze JOB-ID 
# Produces a long ouput. 
# The following lines are part of the output regarding the job requirements.

The Requirements expression for your job reduces to these conditions:

         Slots
Step    Matched  Condition
-----  --------  ---------
[0]       10674  TARGET.Arch == "X86_64"
[1]       10674  TARGET.OpSys == "LINUX"
[3]       10674  TARGET.Disk >= RequestDisk
[5]           0  TARGET.Memory >= RequestMemory
[8]       10674  TARGET.HasFileTransfer
```

By looking through the match conditions, we see that many nodes match our requests for the Linux operating system and the x86_64 architecture, but none of them match our requirement for 51200 MB of memory.

> It is possible to try submitting this `request_memory = 2TB` job yourself by walking through the tutorial `tutorial error101`. To do this, first login as usual, and then load the `tutorial error101`.
> 
> ```
> $ ssh username@login0N.osgconnect.net
> 
> $ tutorial error101
> $ cd tutorial-error101
> $ condor_submit error101_job.submit
> ```
> Then, check the job status the normal way using `condor_q jobID`. Notice your job is still idle. Why? Try using `condor_q -better-analyze` to find out! Make sure to check the submit file for this job too!


## Held Jobs

To restrict your view to all jobs that are currently being held in the queue, use `condor_q userID -hold` or for a single job you can alternatively use `condor_q jobID -hold`. The output of these commands will provide the reason a job was held. 

If you have a large number of held jobs (>10), it can be better to use `condor_q userID -af HoldReason | sort | uniq -c`. The output of this command will output the number of jobs held for different reasons. Jobs are held if there is an issue that the user needs to fix in order for the job to complete successfully. Held jobs loose all progress and files cannot be recovered. 

Jobs can be held for a variety of reasons, including: 
* exceeding request_disk (solution: request more disk space in the submit file and resubmit the job)
* exceeding request_memory (solution: request more memory in the submit file and resubmit the job )
* exceeding run time limit
* file/directory not found (solution: check the names and file paths of all files, make sure all relevant files are listed in `transfer_input_files`)
* /home or /public directories over quota

A more through investigation into a job's attributes that may be causing it to be held can be carried out using the "long" flag, `-l`, shown below. 

## Fixing Held Jobs

Job attributes in the submit file or in the ClassAd can be edited while the job is in the queue using `condor_edit <User/BatchID/JobID> AttributeName NewValue`. Once fixed, a job can be released using `condor_release <User/BatchID/JobID>`.

Errors, such as those in the executable, that are causing a job to go on hold require a job be removed from the queue using `condor_rm <User/BatchID/JobID>`. Once the error has been corrected, the corrected job can be resubmitted using `condor_submit`. 

## Example Held Job Case Study: File Not Found

In this particular case, a user had this in his or her HTCondor submit file:

`transfer_output_files = outputfile`

However, when the job executed, it went into "Held" state:

```
$ condor_q -analyze 372993.0
-- Submitter: login01.osgconnect.net : <192.170.227.195:56174> : login01.osgconnect.net
---
372993.000:  Request is held.
Hold reason: Error from glidein_9371@compute-6-28.tier2: STARTER at 10.3.11.39 failed to send file(s) to <192.170.227.195:40485>: error reading from /wntmp/condor/compute-6-28/execute/dir_9368/glide_J6I1HT/execute/dir_16393/outputfile: (errno 2) No such file or directory; SHADOW failed to receive file(s) from <192.84.86.100:50805>
```

Let's break down this error message piece by piece:

```
Hold reason: Error from glidein_9371@compute-6-28.tier2: STARTER at 10.3.11.39 failed to send file(s) to <192.170.227.195:40485>
```

This part is quite cryptic, but it simply means that the worker node where your job executed (`glidein_9371@compute-6-28.tier2 or 10.3.11.39`) tried to transfer a file to the OSG Connect login node (`192.170.227.195`) but did not succeed. The next part explains why:

```error reading from /wntmp/condor/compute-6-28/execute/dir_9368/glide_J6I1HT/execute/dir_16393/outputfile: (errno 2) No such file or directory```

This bit has the full path of the file that HTCondor tried to transfer back to the Access Point. The reason why the file transfer failed is because `outputfile` was never created on the worker node. Remember that at the beginning we said that the user specifically requested `transfer_outputfiles = outputfile`! HTCondor could not complete this request, and so the job went into Held state instead of finishing normally.

It's quite possible that the error was simply transient, and if we retry, the job will succeed. We can re-queue a job that is in Held state by using condor_release:

```condor_release JOB-ID```


# Find where a Job Ran on the OSPool

Sometimes, it is possible that your jobs will not run successfully at certain sites due differences in hardware and configuration between different institutions. If you notice some or many of your jobs are not running successfully at an institution,  [contact a Research Computing Facilitator](https://support.opensciencegrid.org/support/solutions/articles/12000084585-get-help-). 

To find out where and how many of your jobs are running at different locations in the OSPool, we can use the ClassAd attribute `MATCH_EXP_JOBGLIDEIN_ResourceName` with either `condor_q` or `condor_history`. An example of this command that investigates where 100 jobs a user submitted are currently running may look like `condor_q <batchID, jobID, Username> -limit 100 -af MATCH_EXP_JOBGLIDEIN_ResourceName | sort | uniq -c`. By using -limit 100, this command limits the output to 100 jobs, but this value can be replaced by a higher or lower number depending on the number of jobs you have submitted. 

The output of this command looks like: 

```
      9 Crane
      4 LSU-DB-CE1
      4 ND-CAML_gpu
     71 Rice-RAPID-Backfill
      2 SDSC-PRP-CE1
      6 TCNJ-ELSA
      1 Tufts-Cluster
      3 WSU-GRID
```

In this example, 71 jobs ran at Rice University (Rice-RAPID-Backfill) while only one job ran at Tufts University (Tufts-Cluster). 


# Resources to Investigate Jobs: Standard Error, Standard Output, and HTCondor Log Files

## Log File (`.log`)
A log file is automatically created by HTCondor to track job progress when `log =` is set in the submit file. It captures information such as when a job started, where it ran, the resources it used, reasons for interruption, and exit status codes. 

The top of most `.log` files contain information similar to the following: 
```
/scratch.local/condor/execute/dir_26929/glide_WcAKcv/condor_job_wrapper.sh: line 373: singularity_path_in_cvmfs: command not found
/scratch.local/condor/execute/dir_26929/glide_WcAKcv/condor_job_wrapper.sh: line 396: singularity_path_in_cvmfs: command not found
INFO  Discarding path '/hadoop'. File does not exist
INFO  Discarding path '/ceph'. File does not exist
INFO  Discarding path '/hdfs'. File does not exist
INFO  Discarding path '/lizard'. File does not exist
INFO  Discarding path '/mnt/hadoop'. File does not exist
INFO  Discarding path '/mnt/hdfs'. File does not exist
```
The standard output above is relevant to the inner workings of HTCondor jobs, but should not be concerning to the user. These messages may change subtly depending on the job. For example, a python job submitted to the OSPool may have additional messages such as: 

```
INFO  GWMS Singularity wrapper: PYTHONPATH is set to /cvmfs/oasis.opensciencegrid.org/mis/osg-wn-client/3.5/3.5.18/el7-x86_64/usr/lib/python2.7/site-packages:/cvmfs/oasis.opensciencegrid.org/mis/osg-wn-client/3.5/3.5.18/el7-x86_64/usr/lib64/python2.7/site-packages outside Singularity. This will not be propagated to inside the container instance.
```
A majority of these messages can be ignored, but if you are unsure of whether these messages indicate trouble with your job, please reach out to a Research Computing Facilitator. 

## Error File

Additionally, an error file is automatically created by HTCondor to record standard errors (stderr) the job encounters when `error =` is set in the submit file. If a job experiences an error that prevents it from completing, it will likely be detected by HTCondor and placed on hold. 

## Terminal Output File 

Lastly, HTCondor also saves all standard output (stdout) for a job when `output =` is set in the submit file. This file captures any text printed to the terminal, including output and errors.


# Resources to Investigate Jobs: Job Attributes

Both `condor_q` and `condor_history` can use the "long" flag `-l` to print out a job's attributes. More information about using `-l` to toubleshoot jobs can be found in our "Monitor and Review Jobs with `condor_q` or `condor_history`" guide.

Job attributes commonly used to diagnose issues with jobs include: 

| Attribute      | Description |
| ----------- | ----------- |
| MemoryUsage   | Maximum memory that a job used in MB |
| DiskUsage   | Maximum disk space that a job used in KB |
| BatchName      | Job batch label       |
| RemoteHost   | Location where a job is running |
| ExitCode   | Exit code of a job upon its completion |
| HoldReason   | Human-readable message as to why a job was held. It can be used to determine if a job should be released or not. |
| HoldReasonCode   | Interger value that represents why a job was put on hold |
| JobNotification   | Integer indicating when the user should be emailed regarding a change of status for their job |
| RemotePool   | Name of the pool in which a job is running |
| NumRestarts   | Number of restarts carried out by a job |

Many additional attributes are provided by HTCondor to learn about your jobs, including attributes dedicated to workflows that utilize DAGman and containers.

For more information about these and other attributes, please see the [HTCondor Manual](https://htcondor.readthedocs.io/en/latest/classad-attributes/job-classad-attributes.html)
 
