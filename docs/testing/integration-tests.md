# Integration Tests

This page contains information about integration tests for VPP Agent. The following text describes how to run robot test suites (locally or in any other environment).

---

### VM(s) -ubuntu:
  - required: python, robotframework, docker

  - install Docker CE: https://docs.docker.com/install/linux/docker-ce/ubuntu/#prerequisites
  - install other:
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install git
sudo apt-get install python
sudo apt-get install python3
sudo apt-get install python-pip
sudo apt-get install python-paramiko
sudo pip install requests
sudo pip install robotframework
sudo pip install robotframework-sshlibrary
sudo pip install -U robotframework-requests
sudo pip install --upgrade robotframework-httplibrary
```

  - SSH connection between VMs:
```
sudo apt-get install openssh-server
ssh-keygen
ssh xxx@192.168.100.XX
```

### Clone git vpp-agent:
```
git clone https://github.com/ligato/vpp-agent.git
```

### Download docker images:
```
docker pull ligato/dev-cn-infra:latest             
docker pull ligato/prod_sfc_controller:latest          
docker pull quay.io/coreos/etcd:v3.0.16             
docker pull ligato/dev_sfc_controller:latest 
docker pull ligato/vpp-agent:dev 
docker pull ligato/dev-vpp-agent:dev
```

### Setup local variables:
      - .../vpp-agent/tests/robot/variables/jozo_local_variables.robot

        - IP- VM where you will run tests
        - USER/PASS on this VM
        - Image witch you will use

      - .../vpp-agent/tests/robot/variables/common_variables.robot
        -  If you have docker without sudo, then you need change "sudo docker" on line 7, to "docker"

### Make directory on VM:
```
sudo mkdir /run/vpp
```

### Run test:
  - Change directory to folder with tests:
```
cd .../vpp-agent/tests/robot/suites/crud/
```
  - Run test
 ```
pybot --loglevel TRACE -v VARIABLES:jozo_local loopback_crud.robot
```

### Test results:
      - robot log file, ...:
        - .../vpp-agent/tests/robot/suites/crud/
      - logs: vpp-agent, etcd, ...:
        - .../vpp-agent/tests/robot/suites/crud/results
