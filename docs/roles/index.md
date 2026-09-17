Roles define a set of common node setup operations that can be generalized and deployed to multiple nodes in the same fashion. 

A role usually can define the following subfolders:

- defaults (expects a file 'main.yml')
- handlers (expects a file 'main.yml')
- meta  (expects a file 'main.yml')
- tasks  (expects a file 'main.yml')
- templates


## Defaults
Defaults is used to define default variables used during the role build e.g. the version and options with which slurm is installed on the node.

## Handlers
Handlers are task that only run when called upon by another task. 

## Meta
You can define dependency on other roles here so that this role is executed prior.

## Tasks

This is where the heavy lifiting of the role lives. 