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

Before continoing, explanation of inventory
-------------------------------------------

We are using a mix of dynamc and static inventory. In ansible an inventory is the
place where you put yout hosts (or targets) based on groups and groups of groups, etc

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

This is very flexible becuase we could target one server, one group, a group of...

On the other side with services like AWS, Digital ocean, Linode... this servers
are dynamically created, destroyed, moved... and then there are not in the same 
place, so we need something that will populate this host files.

So we have to use also dynamic invetory, this works running an script (depends on the service)
that gathers data of your server account and then assigns data so ansible could use it

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

# Alias to call easily the environment
[production:children]
*.production.*

```

So afterwards, when we call ansible we will call with the digital ocean
inventory script along with the host file, this means that in the playbooks as
we refer to the hosts exclusively ansible will know where are this servers.


> Use `--list-hosts` before doing anything!

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

now the targe are only the production servers


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
ansible-playbook ./deploy.yml  -i inventory --limit=*stage*
```

Further reading: http://docs.ansible.com/intro_patterns.html


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

