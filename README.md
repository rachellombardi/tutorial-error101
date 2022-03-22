[title]: - "Troubleshooting Job Errors"
[TOC]

# Overview
In this lesson, we'll learn how to troubleshoot jobs that never start or fail in unexpected ways. 

# Troubleshooting techniques

## Diagnostics with condor_q

The `condor_q` command shows the status of the jobs and it can be used 
to diagnose why jobs are not running. Using the `-better-analyze` flag 
with `condor_q` can show you detailed information about why a job isn't 
starting on a specific pool. Since OSG Connect sends jobs to many places, we also need to 
specify a pool name with the `-pool` flag.                              

Unless you know a specific pool you would like to query, checking the `flock.opensciencegrid.org` pool is usually a good place to start.

	$ condor_q -better-analyze JOB-ID -pool POOL-NAME

Let's do an example. First we'll need to login as usual, and then load the tutorial *error101*.

	$ ssh username@login.osgconnect.net
	
	$ tutorial error101
	$ cd tutorial-error101
	$ condor_submit error101_job.submit 

We'll check the job status the normal way:

	condor_q username

For some reason, our job is still idle. Why? Try using `condor_q
-better-analyze` to find out. Remember that you will also need to
specify a pool name. In this case we'll use `flock.opensciencegrid.org`:

	$ condor_q -better-analyze JOB-ID -pool flock.opensciencegrid.org
	 
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

By looking through the match conditions, we see that many nodes match our requests for the Linux operating system and the x86_64 architecture, but none of them match our requirement for 51200 MB of memory. 

Let's look at our submit script and see if we can find the source of this error:

	$ cat error101_job.submit 
	Universe = vanilla
	
	Executable = error101.sh
	
	# to sleep an hour
	Arguments = 3600
	
	request_memory = 2 TB
	
	Error = job.err 
	Output = job.out 
	Log = job.log 
	Queue 1 

See the `request_memory` line? We are asking for 2 Terabytes of memory, when we meant to only 
ask for 2 Gigabytes of memory. Our job is not matching any available job slots because 
none of the slots offer 2 TB of memory. Let's fix that by changing that line to read `request_memory = 2 GB`.

	$ nano error101_job.submit

Let's cancel our idle job with the `condor_rm` command and then resubmit our edited job:

	$ condor_rm JOB-ID
	$ condor_submit error101_job.submit

Alternatively, you can edit the resource requirements of the idle job in queue:

	condor_qedit JOB_ID RequestMemory 2048


## Held jobs and condor_release

Occasionally, a job can fail in various ways and go into "Held"
state. Held state means that the job has encountered some error, and
cannot run. This doesn't necessarily mean that your job has failed, but,
for whatever reason, Condor cannot fulfill your request(s).

In this particular case, a user had this in his or her Condor submit file:

	transfer_output_files = outputfile

However, when the job executed, it went into Held state:

	$ condor_q -analyze 372993.0
	-- Submitter: login01.osgconnect.net : <192.170.227.195:56174> : login01.osgconnect.net
	---
	372993.000:  Request is held.
	Hold reason: Error from glidein_9371@compute-6-28.tier2: STARTER at 10.3.11.39 failed to send file(s) to <192.170.227.195:40485>: error reading from /wntmp/condor/compute-6-28/execute/dir_9368/glide_J6I1HT/execute/dir_16393/outputfile: (errno 2) No such file or directory; SHADOW failed to receive file(s) from <192.84.86.100:50805>

Let's break down this error message piece by piece:

	Hold reason: Error from glidein_9371@compute-6-28.tier2: STARTER at 10.3.11.39 failed to send file(s) to <192.170.227.195:40485>

This part is quite cryptic, but it simply means that the worker node
where your job executed (glidein_9371@compute-6-28.tier2 or 10.3.11.39)
tried to transfer a file to the OSG Connect login node (192.170.227.195)
but did not succeed. The next part explains why:

	error reading from /wntmp/condor/compute-6-28/execute/dir_9368/glide_J6I1HT/execute/dir_16393/outputfile: (errno 2) No such file or directory

This bit has the full path of the file that Condor tried to transfer back to `login.osgconnect.net`. The reason why the file transfer failed is because `outputfile` was never created on the worker node. Remember that at the beginning we said that the user specifically requested `transfer_outputfiles = outputfile`! Condor could not complete this request, and so the job went into Held state instead of finishing normally.

It's quite possible that the error was simply transient, and if we retry, the job will succeed. We can re-queue a job that is in Held state by using `condor_release`: 

	condor_release JOB-ID 


## Retries with periodic_release

It is important to consider that the Open Science Pool is a
very heterogenous environment. You might have a job that works at 95% of
remote sites, but inexplicably fails elsewhere. What to do, then?

Fortunately, we can ask Condor to check if the job failed by looking at
its exit code. If you are familiar with UNIX systems, you may be aware
that a successful program returns "0". Anything other than 0 might be
considered a failure, and so we can ask Condor to monitor for these and
retry if it detects any such failures.

This can be accomplished by adding the following lines to your submit file:

	# Send the job to Held state on failure. 
	on_exit_hold = (ExitBySignal == True) || (ExitCode != 0)  
	
	# Periodically retry the jobs every 10 minutes, up to a maximum of 5 retries.
	periodic_release =  (NumJobStarts < 5) && ((CurrentTime - EnteredCurrentStatus) > 600)

