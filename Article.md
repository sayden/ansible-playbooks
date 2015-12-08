Continuous Delivery with Jenkins, Docker and Ansible
===

I like the DevOps philosophy as I think that every developer must have a broad knowledge of the ecosystem around him/her. In this article I will show how I tested a Continuous Delivery ecosystem for my personal FullStack projects.

**Warning:** This is not a tutorial and is not definetely easy for people who doesn't have a initial understanding con Continuous Integration with Jenkins, Docker or Ansible (althought this last could be covered with knowledge in Puppet, Chef or any other orchestator).

**More Warning** I deserve some pain because I'm not concerned about security in this article. Yes, I know, I'll drink beer until I get a hangover as punishment xD

It's going to be fragmented in 3 parts:
* Part one: The Nodejs project and making a Docker with it in our machine
* Part two: Automating redeploy of the Docker container in a remote machine
* Part three: Redeploy using Jenkins and Ansible when our master branch has changed

# Part one: The Nodejs project and making a Docker with it in our machine

## The Nodejs Project

I have simplified the project to the maximum and it's going to be a very simple Node project that simply shows a message on the screen on plain text. The "projet" is available in https://github.com/sayden/simplest-express-server

### Installing Docker on our machine

First we have to install Docker in our machine for the first tests:

```bash
# Red had based distribution
sudo yum install -y docker

# Debian based
sudo apt-get install -y docker
```

We'll use an image from docker.io called `docker.io/node:4` which refers to the `4.2.3` LTS version. The Dockerfile has this "base" container and will simply `git clone` a repo an expose the 3000 port (the one in use for this specific "app")

```dockerfile
FROM    node:4

# Arguments
ENV REPO="https://github.com/sayden/simplest-express-server.git"
ENV DEST=/srv/node-server
ENV APP=${DEST}/server.js

# Ensure git is installed
RUN apt-get install -y git

# Clone the github repo
RUN git clone ${REPO} ${DEST}

# Go to cloned folder
RUN cd ${DEST} && npm install

# Expose app port
EXPOSE 3000

# Launch app
CMD ["sh", "-c", "cd ${DEST} && npm start"]
```

### Building Docker image

So let's build the image.

```bash
# As root
docker build -t mariocaster/node-server .
```
> Take a look at the last "." in the command as it points to the folder with the previous Dockerfile

### Running Docker image

Once we have the image build and we can see it using `sudo docker images` it's time to run it

```bash
# As root
docker run -d -p 41600:3000 mariocaster/node-server
```

> With `-p 41600:3000` we are telling Docker that if we access in our machine to the 41600 port it will redirect us to the 3000 in the container, the one with the node server

## Part two: Automating redeploy of the Docker container in a remote machine

### Installing Docker on remote host

I'm going to use a VirtualBox virtual machine with Centos 7 installed running on 192.168.1.39 in my case. This machine is going to be called ***THE_HOST***.

First we need to get password-less access to the machine so I use `ssh-copy-id` to pass a public key in my `~/.ssh` folder to *THE_HOST*. I have a handy bash script to gain access because I never remember the exact syntax:

#### BONUS: Gaining SSH access to the machine.

```bash
#!/bin/bash

# gain-ssh-access
echo -e "\n"

if [ "$1" = "-i" ] ; then
  echo "Using interactive mode"

  echo -e "Write the name of the remote user: \c"
  read user

  echo -e "Write the host Ip or name: \c"
  read host

  echo "A public key from ~/.ssh/id_rsa.pub will be used"
  echo "Remote machine will probably ask for permissions password"

  ssh-copy-id -i ~/.ssh/id_rsa.pub $user@$host
else
  echo "You can use interactive mode with -i flag"
  echo "Use ssh-copy-id command if not. example:"
  echo "ssh-copy-id -i ~/.ssh/id_rsa.pub user@host"
fi

echo -e "\n"
```

Once we have access, I have another "handy" Ansible Playbook to install Docker on *THE_HOST*. The script is the following:

```yaml
# add-docker.yml
- host: all
  become: yes
  become_method: sudo

  tasks:
  - name: Add Docker
    yum: name=docker state=present

  - name: Add python-docker-py
    yum: name=python-docker-py state=present

  - name: Docker service must be started
    service: name=docker state=started

```

Of course, in my ansible `hosts` file I have the proper user and password configuration for the machine

```bash
# hosts

[local]
192.168.1.39      ansible_sudo_pass=osboxes.org    ansible_ssh_user=osboxes
```

So I can launch the command like this:

`ansible-playbook -i hosts add-docker.yml`

### Adding the Dockerfile and launching it

We have an Ansible task to add the Dockerfile to *THE_HOST*. Then we have a Playbook that will copy it and build or restart the container.

```yml
- hosts: all
  become: yes
  become_method: sudo
  vars:
    # For handlers/restart-docker.yml
    http_port: 3000
    host_port: 41600

    # Shared here and in handlers/restart-docker.yml
    image_name: mariocaster/node-server

    # Dockerfile to use and destination in target's machine
    source_dockerfile_dir: /path/to/Dockerfile
    docker_dest_dir: /srv/docker

    git_repo: https://github.com/sayden/simplest-express-server.git

  tasks:
    - name: Copy Dockerfile to server
      copy: dest={{ docker_dest_dir }} src={{ source_dockerfile_dir }}

    - name: Re/build Docker image
      docker_image: name={{ image_name }}
                    path={{ docker_dest_dir }}
                    nocache=yes
                    state=build
      notify: Restart Docker image

  handlers:
    - name: Restart Docker image
      docker: name=node ports={{ host_port }}:{{ http_port }} image={{ image_name }} state=reloaded
```

Ok, very easy. Copy the Dockerfile, builds the image and restart (or start for the first time) the container.

If we access in our machine to `0.0.0.0:41600` we'll see our server running.

### Redeploying with one line

We can reuse our Ansible Playbook to redeploy the server as many times as we want, we must simply run the same Playbook again. You can try, if you are using your own git repo, to push some change and launch the same script again to see how "magically" changes.

# Part three: Redeploy using Jenkins and Ansible when our master branch has changed

Ok, now we can redeploy as many times as we want. So now we must configure Jenkins to redeploy the container every time it finds (via polling) any change in the git repo.

There's nothing very special in the Jenkins job. We'll simply execute the redeploy Ansible Playbook. As there are thousands of tutorials about how to trigger Jenkins on Git push (via polling or git hook) I'll not enter on it but I leave here a handy tutorial about it: http://www.nailedtothex.org/roller/kyle/entry/articles-jenkins-gittrigger

Also, you can use the Jenkins Ansible Plugin if you want https://wiki.jenkins-ci.org/display/JENKINS/Ansible+Plugin but I have to say that I feel very comfortable with bash scripts and I usually prefer to write my own scripts so here's the one I used to redeploy the container:

```bash
export ANSIBLE_PLAYBOOKS=/var/local/jenkins

ansible-playbook -vvvv ${ANSIBLE_PLAYBOOKS}/rebuildDocker.yml -i ${ANSIBLE_PLAYBOOKS}/hosts
```

> Needless to say that `jenkins` user must have ssh access too to THE_HOST as well as to the Ansible Playbook and Ansible hosts file

Notes about this last part: I don't enter too deep into Jenkins configuration because, as I said at the beginning of the article, users that are trying to achieve Continous Delivery in their project must have some background knowledge in advance about some tools as this is not a "tutorial" about a tool but more like a "full solution" using various tools available (not necessarly the best).

ansible-playbooks
===

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
