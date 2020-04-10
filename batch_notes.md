#Notes from working with AWS Batch

All managed instances with the default AMI are provisioned with an 8GB EBS drive mounted.

##Under the hood

##Batch Event Sourcing

##Gotchas
These are some of the edge-case errors I've found while testing various load patterns on AWS Batch.

#### CannotInspectContainer, Device Out of Space errors  

Error result:
 - CannotInspectContainer errors: some containers may complete processing but be set to a FAILED state in the AWS system
 - Device Out of Space errors: containers will fail to provision to a container instance and will not run at all

These occur in particular situations where a large number of jobs are executing over a relatively short period of time, 
often with a number of separate batches of job requests being made. While this can be a difficult scenario to reproduce, 
understanding the related factors can make these situations much less likely to occur.  

The factors that seem to cause these types of situations to occur more often are the following:  

- Large batches of jobs which each individually execute quickly
- Underprovisioning of EC2 instances by the Batch scheduler. Seems to occur when multiple large batches of `submit_job()` requests 
are made in relatively short succession, but separated enough that resources have already begun to be provisioned for the previous request. 
For some reason these kinds of situations seem to not kick the auto-scaling group into gear to start provisioning more EC2s to handle 
the subsequent requests
- Provisioning of large instances relative to job size, allowing for large numbers of jobs to be run in parallel  

*But what's happening, eh?*  

When large instances are provisioned (say an r5.24xlarge @ 96 vCPU / 768GB) to run jobs with relatively small resource requirements
(say 1 vCPU / 8GB), the Batch scheduler will automatically parallelize ~96 of those jobs at a time. The instance will continue
grabbing more tasks from the job queue as long as any remain. In cases where instances are underprovisioned (perhaps you tricked
the scheduler into thinking it can chug through the queue with existing EC2 resources faster than it can provision new resources) 
an individual EC2 instance might end up running hundreds of jobs.  Each time an EC2 instance runs a job, it pulls a fresh copy of
the associated docker image and stores it on disk, along with any associated docker daemon logs and configurations. The 
`ecs-agent` which manages the docker images on your EC2 Batch worker instances eventually will clean up these images, but 
by default this happens only every 3 hours. So in the scenario above, an EC2 worker instance might pull hundreds of images, 
which can result in either of the two situations that started this section, depending on your configuration.  Essentially 
this can occur any time EC2 Batch workers stay online for a long period of time while processing large amounts of quickly-executing jobs.

*And how can I keep this from happening to me?*  

If you are using a default Batch AMI, you only have a 8GB EBS hard drive to write images onto.  In this case I've often seen
`Device Out of Space` errors. By burning a custom AMI (from an ecs-optimized base AMI) with a larger EBS volume, I've been 
able to get more mileage out of my provisioned Batch EC2 workers by giving them more space to hold images. If your container
processes write files to disk that aren't needed post-execution, it can help to delete those before you exit your process as well.

 

Some rough example numbers are requesting 200 jobs at 1 cpu / 8 GB each,
5 times spaced out over ~10 minutes, with each job taking less than 4-5 minutes to execute   
