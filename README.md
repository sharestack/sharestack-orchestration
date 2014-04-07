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
from monitoring, logs... to the sharestack application itself.

Actually is designed for 2 aims:

* Creation: Creates the instances dynamically without the need to
touch a web administration tool

* Provision: Installation of all the stack in those provisioned machines

* Deployment: Installs the applications

---

Before continuing, explanation of inventory
-------------------------------------------

We are using a mix of dynamic and static inventory. In ansible an inventory is the
place where you put your hosts (or targets) based on groups and groups of groups

So I could set a host file like this:

```yaml
[atlanta]
host1
host2

[raleigh]
host2
host3

[southeast:children]
atlanta
raleigh

[usa:children]
southeast
northeast
southwest
northwest
```

This is very flexible becuase we could target one server, one group, a group of groups...

On the other side with services like AWS, Digital ocean, Linode... this servers
are dynamically created, destroyed, moved... and this changes their address eventually,
so we need something that will populate this host files with this dynamic adresses.

So we have to use also dynamic invetory, this works running an script (depends on the service)
that gathers data of your server account and then assigns data so ansible could use it, this script
is developed by ansible ([check the list](https://github.com/ansible/ansible/tree/devel/plugins/inventory)) and we have to put in the directory where the hosts are (in our case: `inventory`)

wrapping up, we will put the host names (as if there where groups) assigned in the creation of our instances
 in the static inventory for example:

```yaml
# The groups will be single, will work like an alias for referencing the right address
[1.web.production.sharestack]
[2.web.production.sharestack]
[1.db.production.sharestack]
[2.db.production.sharestack]
[1.lb.production.sharestack]
[1.monitoring.production.sharestack]
[1.logs.production.sharestack]
[1.searchers.production.sharestack]

# The groups

[webservers:children]
1.web.production.sharestack
2.web.production.sharestack

[dbservers:children]
1.db.production.sharestack

[lbservers:children]
1.lb.production.sharestack

[cacheservers:children]
1.cache.production.sharestack

[monitoring:children]
1.monitoring.production.sharestack

[logs:children]
1.logs.production.sharestack

[searchservers:children]
1.searchers.production.sharestack
```

Afterwards, when we call ansible we will call with the digital ocean
inventory script along with the host file (this is done automatically by ansible), 
this means that in the playbooks as we refer to the hosts exclusively ansible
will know where are this servers.


> Use `--list-hosts` before doing anything and check that are the correct hosts!

An example of host patterns

Imagine that we have a directory named `inventory` where our host files are
(yes we can have more than one and combine then when calling)

like this:

```
./inventory
├── production
└── stage
```

File `inventory/production`

```
[1.web.production.sharestack]
web1.sharestack.org
[1.db.production.sharestack]
db1.sharestack.org

[webservers:children]
1.web.production.sharestack

[dbservers:children]
1.web.production.sharestack
```

File `inventory/stage`

```
[1.web.stage.sharestack]
web1.stage.sharestack.org
[1.db.stage.sharestack]
db1.stage.sharestack.org

[webservers:children]
1.web.stage.sharestack

[dbservers:children]
1.web.stage.sharestack
```

Lets check our patterns, our first one is calling all with the pattern `all` or `*`

```bash
$ ansible all -i inventory --list-hosts 
    web1.stage.sharestack.org
    db1.sharestack.org
    web1.sharestack.org
    db1.stage.sharestack.org
```

Now the targe are only the production servers


```bash
$ ansible *production* -i inventory --list-hosts 
    db1.sharestack.org
    web1.sharestack.org
```

Now the staging ones

```bash
$ ansible *stage* -i inventory --list-hosts 
    web1.stage.sharestack.org
    db1.stage.sharestack.org
```

Now the web servers only

```bash
$ ansible webservers -i inventory --list-hosts 
    web1.sharestack.org
    web1.stage.sharestack.org
```

Now production webservers. As we have only 2 we could do this with 2 patterns
production and webserver, or, webserver and not stage

```bash
$ ansible *production*:\&webservers -i inventory --list-hosts 
    web1.sharestack.org
$ ansible \!*stage*:\&webservers -i inventory --list-hosts 
    web1.sharestack.org
```

> Note: the scaping for characters like `&` and `!` because of bash

As we see this is very flexible when targeting deployments, for example only
deploy in stage.

Of course this patterns are used the same way in the playbooks (inside the yaml files)
or with the ansible-playbook command passing the flag `--limit`:

```
$ ansible-playbook ./deploy.yml  -i inventory/  --list-hosts --limit=*production*

playbook: ./deploy.yml

  play #1 (all): host count=4
    178.223.181.38
    178.223.181.129
    85.85.22.45
    178.221.177.110

```

Further reading: http://docs.ansible.com/intro_patterns.html


Creation
-------------

The first step is to create our infrastructure, because we don't have machines
to install the stuff! 
This step will be executed **locally** because we don't have target hosts, only 
Digital ocean service API. Lets setup stuff first.

###Setup

You will need some env vars setted (`DO_API_KEY` & `DO_CLIENT_ID`)in your
system before start with creation block, for example:

```bash
export DO_API_KEY=43buuicn2dundcndnoqp09029393dbdn
export DO_CLIENT_ID=0230ehdfdcnoq292939ndnsdnc272000
```

###Add ssh keys to Digital ocean

Our instances will need ssh keys, in this step we will store this ssh keys in
our digital ocean account so we can access to them when creating the instances
(next step). If the ssh keys are already there doesn't matter :)

First Edit the file (`roles/creation/vars/sshkeys.yml`) with the ssh keys that
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
ansible-playbook ./creation.yml  -i 'localhost,'  --connection=local  --tags="ssh"
```

> **Note**: If you are using a virtualenv to run this as localhost, ansible will look
in the system, so to do this we have to set up to the virtualenv Python
interpreter with this option: `ansible_python_interpreter=~/.virtualenvs/ansible/bin/python`

> so we would end up with:

```
ansible-playbook ./creation.yml  -i 'localhost,'  --connection=local  --tags="ssh" --extra-vars "ansible_python_interpreter=~/.virtualenvs/ansible/bin/python"
```

###Create the instances

toc reate the instances first we have to configure how many instances and which
are theyr settings. to do this edit `roles/creation/vars/instances.yml`. You
only have 2 settings requirements per instance: `name` and `ssh_keys`, the other
ones have defaults in `roles/creation/defaults/main.yml` check them.

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
ansible-playbook ./creation.yml  -i 'localhost,'  --connection=local  --tags="instances,create"
```

Or with virtualenv:

```
ansible-playbook ./creation.yml  -i 'localhost,'  --connection=local  --tags="instances,create" --extra-vars "ansible_python_interpreter=~/.virtualenvs/ansible/bin/python"
```

###Wrapping up

If you want to execute both of them you can use

```
ansible-playbook ./creation.yml  -i 'localhost,'  --connection=local
```

Or with virtualenv:

```
ansible-playbook ./creation.yml  -i 'localhost,'  --connection=local --extra-vars "ansible_python_interpreter=~/.virtualenvs/ansible/bin/python"
```


---

Provision
---------

The provision block will install all the neccessary things so the deployments 
will go perfectly

###Common

This role will create deployments group and user, and install all the common
things like python and git. This will be applied in all the machines

We only have to configure some variables for example in (`inventory/group_vars/all`):

```
sharestack_working_group: "sharestack"
sharestack_working_user: "sharestack"
sharestack_working_user_auth_ssh_keys: 
  - "~/.ssh/id_rsa_sharestack.pub"
```

This varialbes will set up our working user, group and the ssh keys that will
be allowed to log in with that user account

To execute, for example for our local test environment (vagrant in this case)

```
ansible-playbook ./provision.yml -i inventory/ --limit=local --tags="common"
```

###Virtualenv

This role will install virtualenv and create the desired virtualenvs.

First you need to set up some variables (`roles/virtualenv/vars/main.yml`):

```
virtualenv_version: 1.11.4
virtualenv_user: "{{ sharestack_working_user }}"
virtualenv_prefix: /home/{{ virtualenv_user }}/.virtualenvs
virtualenvs_to_create:
  - api
  - core
```

After that you can execute with:

```
ansible-playbook ./provision.yml -i inventory/ --limit=local --tags="virtualenv"
```

###uwsgi

We don't have to setup nothing, execute with:

ansible-playbook ./provision.yml -i inventory/ --limit=local --tags="uwsgi"

###Nginx
<<<<<<< HEAD

ansible-playbook ./provision.yml -i inventory/ --limit=local --tags="nginx"

=======
>>>>>>> d65f0411f5923f823afaedc78f1cbb00381e0300

###Postgres


Deployment
----------

