# ansible-playbooks
A first touch to ansible with some useful playbooks

## Docker playbooks and roles

### Tasks

#### add-docker.yml
Install docker in the target machine

### Handlers

#### restart-docker.yml

* vars:
  * `host_port`: The port to use in the host
  * `http_port`: The port used in the container's app
  * `image_name`: Image name to restart

### Playbooks

#### copy-dockerfile.yml
Copies a Dockerfile to the target's machine.

* vars:
  * `docker_dest_dir`: The destination path to copy Dockerfile to
  * `source_dockerfile_dir`: The path of the Dockerfile in host's machine

#### rebuild-and-launch-docker.yml
Build and runs a Dockerfile in the target's machine


## deployNode.yml
* Checks if server port is open (and opens it if not)
* Clones a git repo
* Install node project dependencies
* Launches PM2 server using maximum cores and **automatic restart on file change**

`ansible-playbook -i hosts -v deployNode.yml`

## gain-ssh-access
Simple script to remember how to install a ssh public key in some host and gain password-less access

## nodeLts.yml
Makes a Node LTS installation using N and adds PM2 and Git


## redeployNode.yml
Redeploys a Node server based on the previous script, basically:
* Pulls git
* Install npm dependencies
* As we turned on --watch on PM2, the server should restart automatically

`ansible-playbook -i hosts -v redeployNode.yml`


## simplePing.yml
The first ansible script to get familiar with ansible syntax.

## common

### tasks

#### addEpelRepo.yml
Adds EPEL repo to any Red Hat based distribution

## hosts
A handy hosts file
