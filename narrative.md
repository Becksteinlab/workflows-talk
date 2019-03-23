# Talk Narrative

## The problem we are trying to solve

At any given time we may have access to multiple disparate compute resources, and many (possibly hundreds) of simulations to execute.
How can we optimally make use of all these resources to perform all this work without driving ourselves (or our students and postdocs) to jump out of a window?

We present an automation solution that has some attractive features:

1. The work of executing simulations is expressed as a *workflow*: a set of discrete tasks with dependencies between them.
   The work we need to do fits this model pretty well.
2. Each task of the workflow can be executed on a different machine in a different context.
   Tasks that are running simulation code can happen on supercomputer nodes, while analysis tasks can happen on workstations, for example.
3. The system is tolerant to task failures, which *will* happen.
   The tasks for running MD can be written so they are re-runnable with no issue.


This solution has some challenges for deployment we would like to improve upon:

1. It requires deployment of a MongoDB instance for storing the workflows themselves.
2. Requires setup of execution environments on each machine that must execute tasks.
   This must often be done taking into account the details of each machine.
3. Requires an understanding of how the system works, and how communication must occur between components, in order to debug persistent failures.
   This talk seeks to provide that understanding at a high level for getting started.

## A typical molecular dynamics workflow

-- view of workflow figure, showing each piece in turn

A typical molecular dynamics workflow probably consists of at least the following, at least when running a single, independent simulation (e.g. not replica exchange).

1. We first set up the system, perhaps on our own fileserver or workstation.
2. We would then push the files required for execution of our simulation to a remote cluster such as Stampede.
   Typically done via SFTP or SCP, or some variant.
3. We'd submit a script to the queueing system on the cluster for executing the simulation, and at some point that simulation would execute on a compute node.
4. When the simulation job has finished, we might pull the resulting data files from the cluster using our fileserver.
5. We could then run any number of postprocessing or analysis tasks.
   We might also decide whether or not to continue the simulation, which would mean we start a new set of tasks beginning with one like (2).

This set of tasks could be expressed as a *workflow*, which happens to be an *directed, acyclic graph* ("DAG") giving nodes as tasks and arrows for dependencies.
This can be thought of as a flowchart of the work we need to perform.
It turns out that the system of automation we are presenting can perform exactly the tasks laid out in this way, as each task is:
1. Discretely defined. Each task is either yet to be started, running, complete, or failed.
   Because the state of a task is one of these discrete values, it is clearly defined when we can execute a task that is dependent on another.
2. Each task is *idempotent*, meaning that if we were in the middle of a task and it failed for some reason, we could just do that task again and we'd be fine.
   There would be no side effects to re-running a task, save for some resource cost (CPU hours, bandwidth).
   This affords a great deal of fault tolerance, which becomes important with the scale of work we are executing.

We can store this workflow in a central place accessible to all machines that need to see it, such as a webserver.
For the case of the system we are presenting here, this happens to be a MongoDB database.

## Fireworks: a general-purpose workflow automation system


