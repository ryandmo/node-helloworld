# About PDS

Package Distro Search tool (abbreviated as PDS), allows to perform package search across multiple OS distributions in a single UI interface.

# Steps for setting up PDS application on server

### Step 1: Install pre-requisite dependencies.
Note: make sure you are logged in as root user

**For RHEL:**

        yum install python python-setuptools gcc git libffi-devel python-devel openssl openssl-devel cronie
        easy_install pip

**For SLES:**

        zypper install python python-setuptools gcc git libffi-devel python-devel openssl openssl-devel cronie
        easy_install pip

**For Ubuntu:**

        apt-get install python python-pip gcc git


### Step 2: Install Python dependencies libraries

        pip install 'pycparser<=2.14' Flask launchpadlib simplejson


###  Step 3: Checkout the code-base from gitlab, into /opt/ folder

        cd /opt/
        git clone git@github.rtp.raleigh.ibm.com:cinderel-ca/PDS.git

###  Step 4: (Optional) Custom configuration
Update configuration file at /opt/PDS/src/config/config.py for email and other settings.

###  Step 5: Set Environment variables

        export PYTHONPATH=/opt/PDS/src/:$PYTHONPATH
        echo 'export PYTHONPATH=/opt/PDS/src/:$PYTHONPATH' > /etc/profile.d/pds.sh

###  Step 6: Start the Flask server as below

        cd /opt/PDS/src/
        nohup python main.py &

###  Step 7: Verify that the server is up, by running the application in browser
        http://server_ip_or_fully_qualified_domain_name:port_number/pds
        
        Where port is the application port set in config.py. By default its 5000


Application should be now up and running, there is default package info data bundled with the application. 
The bundled data may be old and out-dated so it needs to be freshly updated at a regular interval. So this can be achieved 
by running data generation script bundled with the application. The scripts can be found at:

        cd /opt/PDS/scripts/distro_scripts

Some of the scripts are platform specific like RHEL, SLES. So in order to generate data for the same those scripts need to be 
executed on same platform.

Alternatively we can even setup docker containers for generating data. Below are the steps for setting up data generation using docker.


# (Optional) Advance setup for using docker containers to update package data.

## Step 1: Install docker:

**For RHEL:**

For installing docker refer [Install docker on RHEL](http://www.ibm.com/developerworks/linux/linux390/docker.html)

**For SLES:**

        yum install docker

**For Ubuntu:**

        apt-get install docker

### Step 2: Get the docker image from the repo for ubuntu/debian:
**For RHEL:**

Refer [Creating RHEL Base Images](http://containerz.blogspot.in/2015/03/creating-base-images.html) to create image.

**For SLES:**

Refer [Creating SLES Base Images](https://www.suse.com/documentation/sles-12/singlehtml/dockerquick/dockerquick.html#sec.docker.building_images) 
for creating pre-built images from Repo.

**For Ubuntu/Debian:**

As the data will be directly fetched using launchpadlib API no need for docker containers.

### Step 3: Create docker container for all above distros and version, in running state:

        docker run -m 512M --name <os-name-with-version> -it <image-id> /bin/bash

    e.g. of `<os-name-with-version>` SLES_11, SLES_12, RHEL_6_8, RHEL_7_1, RHEL_7_2
    The above naming convention is needed, as the mapping for fetching data is dependent on same.

### Step 4: Sync the source code from host with the docker container at location /opt/ so that every container has its own copy of PDS code.

        docker cp /opt/PDS <os-name-with-version>:/opt/PDS
        
    e.g. of `<os-name-with-version>` SLES_11, SLES_12, RHEL_6_8, RHEL_7_1, RHEL_7_2    

### Step 5: Update the crontab entry from crontab.txt based on distros in container. 
Enable crontab entry on containers:

**For RHEL 6.8:**

        crontab -e

and paste following line

        40 11 * * *  /opt/PDS/src/scripts/distro_scripts/getPackageInfoRHELUsingYUM.sh RHEL_6_8 &

**For RHEL 7.1:**

        crontab -e

and paste following line

        40 11 * * *  /opt/PDS/src/scripts/distro_scripts/getPackageInfoRHELUsingYUM.sh RHEL_7_1 &
    
**For RHEL 7.2:**

        crontab -e
    
**For SLES 11:**

        crontab -e

and paste following line

        40 11 * * *  /opt/PDS/src/scripts/distro_scripts/getPackageInfoSLESUsingZypper.sh SLES_11 &
    
**For SLES 12/SLES 12-sp1:**

        crontab -e

and paste following line

        40 11 * * *  /opt/PDS/src/scripts/distro_scripts/getPackageInfoSLESUsingZypper.sh SLES_12 &

### Step 6: Update the crontab entry from crontab.txt based on HOST for all distros.

        crontab -e

and paste following line

        40 11 * * 2  /opt/PDS/src/scripts/distro_scripts/fetchDataFromDockerAndArchiveOld.sh RHEL_7_2 &
        40 11 * * 2  /opt/PDS/src/scripts/distro_scripts/fetchDataFromDockerAndArchiveOld.sh RHEL_7_1 &
        40 11 * * 2  /opt/PDS/src/scripts/distro_scripts/fetchDataFromDockerAndArchiveOld.sh RHEL_6_8 &
        40 11 * * 2  /opt/PDS/src/scripts/distro_scripts/fetchDataFromDockerAndArchiveOld.sh SLES_11 &
        40 11 * * 2  /opt/PDS/src/scripts/distro_scripts/fetchDataFromDockerAndArchiveOld.sh SLES_12 &
        40 11 * * 1  python /opt/PDS/src/scripts/distro_scripts/getPackageInfoUsingLaunchpad.py ubuntu &
        40 11 * * 1  python /opt/PDS/src/scripts/distro_scripts/getPackageInfoUsingLaunchpad.py debian &
        40 17 * * 1  python /opt/PDS/src/scripts/distro_scripts/data_verification.py.py &

