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

---

Provision
-------------

The first step is to provision our infrastrcuture, because we don't have machines
to install the stuff! 
This step will be executed **locally** because we don't have target hosts, only 
Digital ocean service API. Lets setup stuff first.

###Setup

You will need some env vars setted (`DO_API_KEY` & `DO_CLIENT_ID`)in your
system before start with provision block, for example:

```bash
export DO_API_KEY=43buuicn2dundcndnoqp09029393dbdn
export DO_CLIENT_ID=0230ehdfdcnoq292939ndnsdnc272000
```

###Add ssh keys to Digital ocean

Our instances will need ssh keys, in this step we will store this ssh keys in
our digital ocean account so we can access to them when creating the instances
(next step). If the ssh keys are already there doesn't matter :)

First Edit the file (`roles/provision/vars/sshkeys.yml`) with the ssh keys that
you want to create, for example:

```yaml
ssh_keys:
  - name: sharestack1
    file: ~/.ssh/id_rsa_sharestack1.pub
  - name: sharestack2
    file: ~/.ssh/id_rsa_sharestack2.pub
  - name: sharestack3
    file: ~/.ssh/id_rsa_sharestack3.pub
```

We could execute only this block ("SSH keys addition") with
(`-i 'localhost,'` is a hack for not use an inventory file):

```
ansible-playbook ./provision.yml  -i 'localhost,'  --connection=local  --tags="ssh"
```

> **Note**: If you are using a virtualenv to run this as localhost, ansible will look
in the system, so to do this we have to set up to the virtualenv Python
interpreter with this option: `ansible_python_interpreter=~/.virtualenvs/ansible/bin/python`

> so we would end up with:

```
ansible-playbook ./provision.yml  -i 'localhost,'  --connection=local  --tags="ssh" --extra-vars "ansible_python_interpreter=~/.virtualenvs/ansible/bin/python"
```

###Create the instances

toc reate the instances first we have to configure how many instances and which
are theyr settings. to do this edit `roles/provision/vars/instances.yml`. You
only have 2 settings requirements per instance: `name` and `ssh_keys`, the other
ones have defaults in `roles/provision/defaults/main.yml` check them.

As example this would be a instance creation setup:

```yaml
instances:
  - name: development.test.1
    image: 1505447
    size: 66
    region: 5
    wait: false
    ssh_keys:
      - sharestack
  - name: development.test.2
    image: 361740
    wait: false
    ssh_keys:
      - sharestack
```

To execute, the same rules as the previous block are applied (local connection,
virtualenv use caution...).

To execute only this block we would use:

```
ansible-playbook ./provision.yml  -i 'localhost,'  --connection=local  --tags="instances,create"
```

Or with virtualenv:

```
ansible-playbook ./provision.yml  -i 'localhost,'  --connection=local  --tags="instances,create" --extra-vars "ansible_python_interpreter=~/.virtualenvs/ansible/bin/python"
```

###Wrapping up

If you want to execute both of them you can use

```
ansible-playbook ./provision.yml  -i 'localhost,'  --connection=local
```

or with virtualenv:

```
ansible-playbook ./provision.yml  -i 'localhost,'  --connection=local --extra-vars "ansible_python_interpreter=~/.virtualenvs/ansible/bin/python"
```


---

Automation
----------

###Configuration
###Deploys

