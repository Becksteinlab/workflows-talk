# Automated workflows

Materials for ACS 2019 talk on automated workflows for molecular dynamics simulations.

## Figures

`workflow.svg`
: This figure is designed to be presented in pieces to explain the core ideas from the perspective of someone who wants to run MD traditionally. Starting from 1) setup, the user 2) pushes files to the compute cluster they have access to, 3) runs a simulation on a compute node, 4) pulls the files back to their infrastructure, then 5) performs various follow-up processes. This exact workflow is what we seek to automate, and these can be modeled as a directed, acyclic graph. If we store this graph in a web-accessible server, in principle each of these steps can be executed by the machines involved, in the right order, by the right machines. The same principle can be used to run hundreds of simulations at once using many compute resources.
