# ansible-playbooks
A first touch to ansible with some useful playbooks

### addEpelRepo.yml
Adds EPEL repo to any Red Hat based distribution

### deployNode.yml
* Checks if server port is open (and opens it if not)
* Clones a git repo
* Install node project dependencies
* Launches PM2 server using maximum cores and **automatic restart on file change**

`ansible-playbook --user=osboxes --ask-sudo-pass -i hosts -v deployNode.yml`

### redeployNode.yml
Redeploys a Node server based on the previous script, basically:
* Pulls git
* Install npm dependencies

As we turned on --watch on PM2, the server should restart automatically

`ansible-playbook --user=osboxes --ask-sudo-pass -i hosts -v redeployNode.yml`

### gain-ssh-access
Simple script to remember how to install a ssh public key in some host and gain password-less access

### nodeLts.yml
Installs minimum required packages for a Node server
* Node from repos (currently 0.10.4)
* NPM
* N (via npm)
* Node LTS (via n, currently 4.2.3)
* PM2 (via npm)
* Git

### simplePing.yml
The first ansible script to get familiar with ansible syntax.

### hosts
A handy hosts file
