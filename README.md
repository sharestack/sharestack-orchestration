sharestack-orchestration
========================

Orchestration stuff of all sharestack infrastructure with Ansible

Requirements
------------

* ansible >= 1.6
* dopy >= 0.2.3

Description
-----------

This big ansible playbook orchestrates all the infrastructure of sharestack,
from monitoring, logs to the sharestack application itself.

Actually is designed for 2 aims:

* Provision: Creates the instances dynamically without the need to
touch a web administration tool

* Automation: Installation of all the stack in those provisioned machines

