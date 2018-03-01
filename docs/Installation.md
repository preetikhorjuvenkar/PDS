# Steps for setting up PDS application on server

The instructions provided below specify the steps for SLES 11 SP4/12/12 SP1/12 SP2 and Ubuntu 16.04/17.04/17.10:

_**NOTE:**_
* make sure you are logged in as user with sudo permissions

### Step 1: Install prerequisite

* For SLES (11 SP4, 12):

        sudo zypper install -y python python-setuptools gcc git libffi-devel python-devel openssl openssl-devel cronie python-xml pyxml tar wget aaa_base which w3m
        sudo easy_install pip
        sudo pip install 'cryptography==1.4' Flask launchpadlib simplejson logging

* For Ubuntu (16.04, 17.04, 17.10):

        sudo apt-get update
        sudo apt-get install -y python python-pip gcc git python-dev libssl-dev libffi-dev cron python-lxml apache2 libapache2-mod-wsgi
        sudo pip install 'cryptography==1.4' Flask launchpadlib simplejson logging

* For SLES (12 SP1, 12 SP2, 12 SP3):

        sudo zypper install -y python python-setuptools gcc git libffi-devel python-devel openssl openssl-devel cronie python-xml pyxml tar wget aaa_base which w3m apache2 apache2-devel apache2-worker apache2-mod_wsgi
        sudo easy_install pip
        sudo pip install 'cryptography==1.4' Flask launchpadlib simplejson logging

* if "/usr/local/bin" is not part of $PATH add it to the path:

        echo $PATH
        export PATH=/usr/local/bin:$PATH
        sudo sh -c "echo 'export PATH=/usr/local/bin:$PATH' > /etc/profile.d/alternate_install_path.sh"

###  Step 2: Checkout the source code, into /opt/ folder

        cd /opt/
        sudo git clone https://github.com/linux-on-ibm-z/PDS.git
        cd PDS

Note: In case PDS code is already checked out, do the following for latest updates

        cd /opt/PDS
        sudo git pull origin master

###  Step 3: Set Environment variables

        sudo sh -c "echo 'export PYTHONPATH=/opt/PDS/src/classes:/opt/PDS/src/config:$PYTHONPATH' > /etc/profile.d/pds.sh"

### Step 4: Install and configure PDS

* SLES (11 SP4, 12):

    #### Copy the init.d script to start/stop/restart PDS application

        sudo chmod 755 -R /opt/PDS/src/setup
        cd /opt/PDS/src/setup
        sudo ./create_initid_script.sh

    #### Enable pds service

        sudo systemctl reload pds

    #### Start the Flask server as below

        sudo service pds start

* SLES (12 SP1, 12 SP2, 12 SP3) and Ubuntu (16.04, 17.04, 17.10):

    #### Copy the apache configuration file from `/opt/PDS/src/config/pds.conf` into respective apache configuration folder as below
    * SLES (12 SP1, 12 SP2, 12 SP3):

            sudo cp -f /opt/PDS/src/config/pds.conf /etc/apache2/conf.d/pds.conf

    * For Ubuntu (16.04, 17.04, 17.10):

            sudo cp -f /opt/PDS/src/config/pds.conf /etc/apache2/sites-enabled/pds.conf
            sudo mv /etc/apache2/sites-enabled/000-default.conf /etc/apache2/sites-enabled/z-000-default.conf

    #### Create new user and group for apache

        sudo useradd apache
        sudo groupadd apache

    #### Enable authorization module in apache configuration(Only for SLES 12 SP1, 12 SP2, 12 SP3)

        sudo a2enmod mod_access_compat

    #### Set appropriate folder and file permission on /opt/PDS/ folder for apache

        sudo chown -R apache:apache /opt/PDS/


    #### Start/Restart Apache service

        sudo apachectl restart

###  Step 5: Verify that the PDS server is up and running

```http://server_ip_or_fully_qualified_domain_name:port_number/pds```

_**NOTE:**_ 

* For SLES (11 SP4, 12) by default the port_number will be 5000
* For SLES (12 SP1, 12 SP2, 12 SP3) and Ubuntu (16.04, 17.04, 17.10)  by default the port_number will be 80

###  Step 6: (Optional) Custom configuration
Following configuration settings can be managed in `/opt/PDS/src/config/config.py`:

        <PDS_BASE> - Base location where PDS is Installed/Cloned. Defaults to `/opt/PDS/`

        <DATA_FILE_LOCATION> - Location of folder containing all distribution specific JSON data
        
        <LOG_FILE_LOCATION> - Location of folder containing PDS logs
        
        <enable_proxy_authentication> - Flag enabling/disabling proxy based network access
        
        <proxy_user> - Proxy server user name
        
        <proxy_password> - Proxy server password
        
        <proxy_server> - Proxy server IP/fully qualified domain name
        
        <proxy_port> - Proxy port number
        
        <DEBUG_LEVEL> - Set Debug levels for the application to log
        
        <server_host> - IP/fully qualified domain name of server where PDS application will be deployed
        
        <server_port> - PDS port on which application will be accessible to end users

        <SUPPORTED_DISTROS> - Mapping of all the supported distros, new distros added need to be mapped here.

        <MAX_RECORDS_TO_SEND> = Max number of records returned to the client. Defaults to 100

        <CACHE_SIZE> - Number of searches to be cached. Default to 10

_**NOTE:**_
* In order to add new distribution support refer [here](Adding_new_distros.md)

In case any of the parameters are updated, the server has to be restarted:

* SLES (12 SP1, 12 SP2, 12 SP3) and Ubuntu (16.04, 17.04, 17.10):

    #### Start/Restart Apache service

        sudo apachectl restart

* SLES (11 SP4, 12):

    #### Start the Flask server as below

        sudo service pds start

# Monitoring PDS application

We have 2 PDS setups running. The details can be seen in the table below. The PDS tool is hosted on docker containers on VMs.


|    PDS		 |				URL			 			 |			VM				 |		Container name          |
| -------------- |  ------------------------------------ | ------------------------- | ---------------------------- |
|  External PDS  |	 http://148.100.110.182:80/pds/ 	 |	148.100.110.182	         |       pds_v143_opensource_live1                       |
|				 |                                       |                           |                              |
|  Internal PDS  |	http://pds.pok.stglabs.ibm.com/pds	 |	ecos0044 (9.47.71.241) & |	pds_live_internal_prod_v143 |
|				 |										 |	ecos0042 (9.47.69.130)	 |	pds_live_internal_prod_v143 |

_**Note:**_ The Internal PDS setup has two containers associated with it. In case the internal PDS is down, one has to check both the containers.

## Actions to be taken if the PDS tool is down.

Ecos0041 and Ecos0017 each have a Jenkins job named "PDS_links" which monitors the status of the PDS tools. These jobs are configured to be run every hour. In case any of the PDS tool is down, the job will fail and a notification sent onto the slack channel. Once a notification is received, perform the following actions:

1. Check the log for the failed Jenkins Job to know which of the PDS link is down.

    The PDS link may be down due to any of the following reasons:
    * The VM hosting the docker container for PDS is down.
    * The docker services on the VM are down.
    * The docker container hosting the PDS is down.
    * The apache services in the docker container hosting the PDS is down.

2. Once the link is known, refer the table above to know the VM and container details. Then perform the following actions:

   * Check if the VM is accessible. If it is not, 
        *  contact your system administrator to get the VM up. 
        *  Perform the next step.
   * Check if the docker services are running on the VM using command `service docker status`.
      If the docker service is down:
        *  Start the docker service using command `sudo service docker start`
        *  Verify its status using `service docker status` 
        *  Perform the next step.
   * Check if the docker container is up. Use command `docker ps ` to see the list of running containers. In case the docker container is down, perform the following steps:
        *  Start the container using command `sudo docker start <container_name>`
        *  Perform the next step.
   * Attach to the container and check if the apache service is up, using command `ps -axf | grep apache2`. In case the apache service is down, perform the following steps:
        *  Attach to the container using `sudo docker attach <container_name>`
        *  Start apache service using command `apachectl restart`
        *  Check if apache services are running using `ps -axf | grep apache2` or `apachectl status`
  
  ## Steps to delete the PDS stats cache and restart PDS
   In order to delete the internal PDS stats cache, perform the following steps:
   * Attach to the PDS container using `sudo docker attach <container_name>`
   * Stop apache services using command `apachectl stop`. Check the status using command `apachectl status`
   * Delete the file `pdslog.db` from `/opt/PDS/stats`.
   * Delete the log file `pds_access.log` from `/opt/PDS/logs`.
   * Start apache service using command `apachectl restart`
   * Check if apache services are running using `ps -axf | grep apache2` or `apachectl status`

