This a vagrant file that installs an allinone with nova-docker on a centos7 machine and fires the services. This job is just a dirty modification of the great job Toni(@celebdor) did here: https://github.com/celebdor/nova_docker_midonet so all the credit should go to him, I just put some extra cheese into the pizza :)

Install vagrant.

http://docs.vagrantup.com/v2/installation/

Add vagrant box:

vagrant box add centos7 https://f0fff3908f081cb6461b407be80daf97f07ac418.googledrive.com/host/0BwtuV7VyVTSkUG1PM3pCeDJ4dVE/centos7.box

clone the repo

install vagrant reload plugin:

vagrant plugin install vagrant-reload

vagrant up
