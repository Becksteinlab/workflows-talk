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

![molecular dynamics workflow](figures/workflow.svg)

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
1. Discretely defined. Each task either waiting for dependencies, ready, running, complete, or failed.
   Because the state of a task is one of these discrete values, it is clearly defined when we can execute a task that is dependent on another.
2. *Idempotent*, meaning that if we were in the middle of a task and it failed for some reason, we could just do that task again and we'd be fine.
   There would be no side effects to re-running a task, save for some resource cost (CPU hours, bandwidth).
   This affords a great deal of fault tolerance, which becomes important with the scale of work we are executing.

We can store this workflow in a central place accessible to all machines that need to see it, such as a webserver.
This system can execute hundreds of these workflows, perhaps each running very different simulations, at the same time, using the same infrastructure.
This is powerful, and can win us some immense throughput.

As an illustration of this throughput, here is the throughput a system like this delivered in late summer 2016.
We used four compute clusters, performing nearly 400 simulations of the membrane protein NapA to compute an alchemical binding pathway for free energy calculation.
Each simulation numbered over 130,000 atoms, and over 15 days we collected over 13 microseconds of trajectory data.
These simulations were executed in short (~4 hour) segments, nearly 20,000 segments in total.
We consumed 1.1 million CPU hours in this time.

![throughput](figures/throughput_panel.png)

## Fireworks: a general-purpose workflow automation system

The system we have described so far is [`fireworks`](https://github.com/materialsproject/fireworks), a workflow automation framework written entirely in Python.
This was not developed by us, but is an open source project born out of *The Materials Project*, a large-scale effort to obtain computed information on known and predicted materials.
This is not the first workflow automation system that uses DAGs for expression of work, and it won't be the last, but it is effective for our needs.

Fireworks includes the following data elements, with fun names to make them easier to talk about:
1. *Workflow*: a DAG, with tasks as nodes and dependencies expressed as vectors between them.
2. *FireWork*: a single node/task in a *Workflow*, with discrete state (waiting, ready, running, complete, failed).
3. *Firetask*: a subtask of a *FireWork*. These are run in sequence within a *FireWork*, and all must succeed for the *FireWork* to succeed.

These data elements are handled by the following infrastructure components:
1. A *Launchpad*, where the Workflows we construct are stored along with the execution states of their tasks.
   This is a MongoDB database.
2. *Fireworkers*: processes launched on other machines to execute work.
   These processes reach out across the network, often the internet, to query the *Launchpad*.
   They will grab the next available *ready* *FireWork*, set its state to *running*, and execute it.
   If it fails to complete, it will set the state to *failed*; if it succeeds it will set it to *complete*.
   These processes must be able to reach the *Launchpad* to work.

To illustrate these components all interacting, we'll return to our example of executing MD on a remote cluster.

![architecture and dataflow of Fireworks](figures/dataflow_architecture.svg)

Fireworkers running on the fileserver and compute nodes query the Launchpad for available work.
For a Workflow that is yet to see execution, FireWorks with no upstream dependencies are in a "ready" state, while those with upstream dependencies are "not ready".

![architecture and dataflow of Fireworks](figures/dataflow_architecture_r.svg)

FireWorks that are "ready" can be executed by a requesting Fireworker; when executing, the FireWork is in a "running" state:

![architecture and dataflow of Fireworks](figures/dataflow_architecture_1.svg)

During this time, other Fireworkers may be constantly requesting work from the Launchpad.
Because no FireWorks are in the "ready" state, they will get nothing to do.
It should be noted, however, that when many workflows such as this are avialble to execute in the Launchpad, there is usually more work available to do.

When a Fireworker completes its work without issue, it changes the state of its FireWork to "complete".
Any FireWorks whose upstream dependencies are all "complete" are set to "ready".

![architecture and dataflow of Fireworks](figures/dataflow_architecture_2r.svg)
