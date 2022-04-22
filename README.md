# neops ansible installer

## Requirements

### On target machine

* Ubuntu 20.04 LTS
* SSH access with pubkey auth with either
  * user with sudo permissions, or
  * direct root access

### On controller

* ansible
* python modules
  * netaddr 
  * passlib 

#### For local testing

* vagrant with virtualbox provider

## Setup

### Initial controller setup

#### Install software

* Ubuntu

```
$ sudo apt install ansible python3-passlib python3-netaddr vagrant virtualbox
```

* Fedora

```
$ sudo dnf install ansible python3-passlib+bcrypt.noarch python3-netaddr.noarch vagrant VirtualBox
```

* Arch Linux

```
$ sudo pacman -S ansible python-netaddr python-passlib vagrant virtualbox
```

#### Code checkout & modules install

```
$ git clone git@github.com:zebbra/neops-deploy
Cloning into 'neops-deploy'...
[...]
$ cd neops-deploy
$ make
Starting galaxy collection install process
[...]
Starting galaxy collection install process
[...]
```

### neops deployment configuration

Create `overrides.yml` file

```
$ cp -p overrides.dist.yml overrides.yml
```

See configuration file for options to set, some are mandatory, e.g. hostnames (`neops_hostname.*`).

All variables in `roles/*/defaults/*.yml` and `inventory/group_vars/all/defaults.ymls` can be overridden, see corresponding files for a list of possible options.

### Target configuration

_(This step is optional if only used locally with vagrant)_

Add target system to inventory file (`inventory/hosts`), e.g

* Direct root access
```
my-target-hostname ansible_host=fqdn-or-ip
```
* Non-root and pw-less sudo
```
my-target-hostname ansible_host=fqdn-or-ip ansible_user=my-sudo-user ansible_become=yes
```
* Non-root and pw-less sudo
```
my-target-hostname ansible_host=fqdn-or-ip ansible_user=my-sudo-user ansible_become=yes
```
* Non-root and sudo with password (consider [setting up vault](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html#tip-for-variables-and-vaults) instead of adding this to inventory in plaintext)
```
my-target-hostname ansible_host=fqdn-or-ip ansible_user=my-sudo-user ansible_become=yes ansible_become_password=my-pw
```

* Local machine deployment (non-ssh)
```
my-target-hostname ansible_connection=local
```

For additional connection options, see [ansible inventory docs](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html).

## Deployment

### Local test environment

```
$ vagrant up
```

provisions a new VM from scratch (use `vagrant destroy` first to force recreate).

To reprovision existing host

```
$ vagrant up --provision
```

### Production environment

#### Test connection to remote host

```
$ ansible -m ping all
ansible -m ping all
neops | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

#### Full deployment

```
$ ansible-playbook neops.yml
PLAY [neops] *******************************************************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************************************
ok: [neops]
[...]
PLAY RECAP *********************************************************************************************************************************************************
neops                      : ok=45   changed=5    unreachable=0    failed=0    skipped=5    rescued=0    ignored=1   
```

#### Only deploy specific components

_NOTE: This will not consider dependencies between components and it may fail, e.g. when deploying containers but docker is not yet installed. Always do an initial full deployment first._

* neops containers
```
$ ansible-playbook -t neops neops.yml
```
* neops and postgres
```
$ ansible-playbook -t neops,postgres neops.yml
```
* all containers
```
$ ansible-playbook -t container neops.yml
```

List all possible tags:

```
$ ansible-playbook --list-tags neops.yml

playbook: neops.yml

  play #1 (all): neops  TAGS: []
      TASK TAGS: [cert, container, data, docker, elasticsearch, etcd, flower, kea, keycloak, kibana, neops, neops-beat, neops-frontend, neops-init, neops-worker, network, pgbouncer, postgres, redis, tftp, traefik, user]
```

#### Common tasks

* Update neops image:
  * Add/Change `neops_image_tag` in `overrides.yml`
  * Reprovision neops `ansible-playbook -t neops neops.yml`
* Scale frontend containers
  * Add/Change `neops_scale_frontend` in `overrides.yml`
  * Reprovision neops `ansible-playbook -t neops-frontend neops.yml`
* Scale worker containers
  * Add/Change `neops_scale_worker` in `overrides.yml`
  * Reprovision neops `ansible-playbook -t neops-worker neops.yml`

#### Using production deployment method for vagrant VM

Add the output of `vagrant ssh-config` to your OpenSSH config, e.g.

```
$ vagrant ssh-config | tee -a ~/.ssh/config
Host neops
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/maetthu/git/zebbra/neops-deploy/.vagrant/machines/neops/virtualbox/private_key
  IdentitiesOnly yes
```

and add `neops` host to the inventory

```
$ echo "neops ansible_user=vagrant ansible_become=yes" > inventory/hosts
```

## Architecture

### Containers

All containers are prefixed with `neops-` by default (see `neops_container_prefix` variable):

```
$ docker ps
CONTAINER ID   IMAGE                                                 COMMAND                  CREATED          STATUS          PORTS                                      NAMES
bb754315c069   quay.io/zebbra/neops-atftpd:latest                    "/app/entrypoint.sh"     9 minutes ago    Up 9 minutes    0.0.0.0:69->69/udp                         neops-tftp
add50a004aa3   quay.io/zebbra/neops-kea-dhcp:latest                  "/app/entrypoint.sh"     10 minutes ago   Up 10 minutes   0.0.0.0:67->67/udp, 8080/tcp               neops-kea
6f5978b34a49   docker.elastic.co/kibana/kibana:7.9.0                 "/usr/local/bin/dumb…"   10 minutes ago   Up 10 minutes   5601/tcp                                   neops-core-kibana
f08ef4b8b222   mher/flower:latest                                    "celery flower --bro…"   10 minutes ago   Up 10 minutes   5555/tcp                                   neops-core-flower
b6de1246bef2   quay.io/zebbra/neops-core:develop                     "/usr/local/bin/cele…"   11 minutes ago   Up 11 minutes   8000/tcp                                   neops-core-beat
d04c9b8f3553   quay.io/zebbra/neops-core:develop                     "/usr/local/bin/cele…"   11 minutes ago   Up 11 minutes   8000/tcp                                   neops-core-worker_2
0f9ef9cedec1   quay.io/zebbra/neops-core:develop                     "/usr/local/bin/cele…"   11 minutes ago   Up 11 minutes   8000/tcp                                   neops-core-worker_1
d13ced6f87a4   quay.io/zebbra/neops-core:develop                     "/usr/local/bin/cele…"   11 minutes ago   Up 8 minutes    8000/tcp                                   neops-core-worker_0
f93fdd8b53a4   quay.io/zebbra/neops-core:develop                     "/usr/local/bin/daph…"   11 minutes ago   Up 11 minutes   8000/tcp                                   neops-core-frontend_2
a2d006aa2ab0   quay.io/zebbra/neops-core:develop                     "/usr/local/bin/daph…"   11 minutes ago   Up 11 minutes   8000/tcp                                   neops-core-frontend_1
d621bff9448f   quay.io/zebbra/neops-core:develop                     "/usr/local/bin/daph…"   11 minutes ago   Up 11 minutes   8000/tcp                                   neops-core-frontend_0
904296b285df   docker.elastic.co/elasticsearch/elasticsearch:7.9.0   "/tini -- /usr/local…"   11 minutes ago   Up 11 minutes   9200/tcp, 9300/tcp                         neops-core-elasticsearch
45dceec4e435   quay.io/coreos/etcd:latest                            "sh -c 'etcd -listen…"   11 minutes ago   Up 11 minutes   2379-2380/tcp                              neops-core-etcd
d4cf4fee7ed2   edoburu/pgbouncer:latest                              "/entrypoint.sh /usr…"   12 minutes ago   Up 11 minutes   5432/tcp, 6432/tcp                         neops-core-pgbouncer
e810e58fb2c0   postgres:11-alpine                                    "docker-entrypoint.s…"   12 minutes ago   Up 12 minutes   5432/tcp                                   neops-core-postgres
bf66dea1674d   redis:5                                               "docker-entrypoint.s…"   12 minutes ago   Up 12 minutes   6379/tcp                                   neops-core-redis
558c5fae3e29   jboss/keycloak:10.0.2                                 "/opt/jboss/tools/do…"   12 minutes ago   Up 12 minutes   8080/tcp, 8443/tcp                         neops-keycloak
7b443c2521f7   postgres:11-alpine                                    "docker-entrypoint.s…"   12 minutes ago   Up 12 minutes   5432/tcp                                   neops-keycloak-postgres
0bfa98007be8   traefik:2.5                                           "/entrypoint.sh --en…"   12 minutes ago   Up 9 minutes    0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   neops-traefik
```

### Data volumes

Located in `/opt/neops` by default (see `neops_container_data` variable):

```
# tree -d -L 2 /opt/neops/
/opt/neops/
├── kea
├── keycloak
│   ├── config
│   ├── export
│   └── postgres
├── neops
│   ├── elasticsearch
│   ├── frontend
│   └── postgres
├── tftp
└── traefik
    └── certs
```

## Maintenance

The  system user (`neops` by default) has permissions to manage docker containers and use sudo and can be used to perform maintenance tasks.

Docker containers can be stopped or restarted as usual, but all local changes managed by will be overwritten in the next deployment run, including
* state of the containers (stopped ones will be started again)
* managed configuration files and container configs/environment
