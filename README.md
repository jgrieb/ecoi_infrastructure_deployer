# README

Vagrant project to automatize the deployment of the DiSSCo digital object repository infrastructure

## 1. Pre-requisites
- Vagrant (https://www.vagrantup.com/downloads.html)
- Rsync (https://www.vagrantup.com/docs/synced-folders/rsync.html). On Windows can be installed with Cygwin (https://cygwin.com/install.html)

## 2. Configuration
1. Copy the file ```config/config_template.json``` as ```config/config.json``` and edit its values to match your deployment.
This file will contain:
    - The passwords to be used when setting up the different applications (cordra, mongodb, elk, etc.)
    - The provider of the virtual machines (aws, virtualbox) and the deployment environment (prod/test/dev, see more below)
    - Credentials to connect to the cloud provider (aws) in "infrastructure->aws". AWS credentials must also be provided in "software->general->aws_access", these should be from an accound that only has the permission to interact with S3 (download Cordra software & upload backup)
    - The internal and external private keys to be installed in the virtual machines, so ansible can run commands on them.    

2. Configuration about the specifications of the different VMs (type, IPs, etc.) are defined in the Vagrant file

3. Configuration of the different applications to be install in the VMs, like what version of the applications we use or on
what ports they will be listening can be edited inside the ansible variables files (eg: ansible/group_vars/all.yml)

4. Copy the cordra_privatekey to be used by the Cordra instance to communicate with the handle server (http://hdl.handle.net/)
into the directory **ansible/roles/cordra/files/cordra/cordra_nsidr_server/**

## 2. Separating production, test and dev environment
Test and production environments are ought to be on AWS, dev environment locally on your machine with the VM software Virtualbox. You need the following configuration:

#### For Production:
```json
{
  "deployment":{
    "environment": "production",
    "domain_prefix": "",
    "default_provider": "aws",
    ...
  }
}
```

#### For test:
```json
{
  "deployment":{
    "environment": "test",
    "domain_prefix": "test.",
    "default_provider": "aws",
    ...
  }
}
```

#### For local dev:
For the dev environment on your local machine make sure you have [Virtualbox](https://www.virtualbox.org/wiki/Downloads) installed.
```json
{
  "deployment":{
    "environment": "test",
    "domain_prefix": "test.",
    "default_provider": "virtualbox",
    ...
  }
}
```
For the local setup to work a host-only network is needed. For this open the Virtualbox application, go to "File -> Host-only network manager" and create an adapter with the name "vboxnet0" and the following configuration:

- IPv4 Address: 172.28.128.1
- IPv4 Mask: 255.255.255.0
- DHCP not enabled

Then, copy the whole content of the file config/local_dev_inventory.ini and use it to replace the content of ansible/inventory.ini. With this you have a default configuration for the IP addresses of the VMs. You can skip the next step and start the setup directly with ```vagrant up```

If any of the configuration above conflicts with your host machine config, you can adjust the IP addresses to your need and update the changes in the Vagrantfile.

#### Note:
After creation of the VMs, Vagrant stores the information related to the instances in the .vagrant/ folder. Therefore, when you change the deployment environment you have to delete the .vagrant/ folder.


## 3. Provision and execution on AWS
1. Modify the files  ```config/config.json```, ```Vagrantfile``` and ```ansible/group_vars/all``` to match your deployment

2. Make sure that in your desired cloud provider you have set up the following aspects and they are available for the credentials that
vagrant will use to connect to that cloud provider:
    - the public key pair configured in the desired ***provider-region***
    - the security groups
    - elastic/public IPs    

3. Open an admin command line console in your machine, here the process is different for production and test

#### Production environment:
1. Go to the folder where the vagrant file is and run the command ```vagrant up --no-provision``` for creating the machines in the cloud provider
and updating the ansible/inventory.ini with their private IPv4 IPs. After that run ```vagrant reload --provision``` so all the provisioners will be executed
including the ansible provisioner inside the cordra_nsidr_server that is responsible to install all the software in the VMs
Note: This Vagrantfile won't work if we try to execut it inside a Linux VM running on Hyper-V on a windows host, because can't change permissions
of private ssh keys


#### Test environment:
The problem is that now Elastic IPs are defined for the test environment and the public IPv4 addresses of the machines change on every reboot. Therefore:

1. Go to the folder where the vagrant file is and run the command ```vagrant up test_monitoring_server test_db_server test_search_engine_server test_ds_viewer_server test_cordra_prov_server``` to create and provision the machines.
2. Run ```vagrant up --no-provision test_cordra_nsidr_server```
3. Gather the private IPv4 addresses of the machines and set their values in ansible/inventory.ini
4. Set manually the routes for test.nsidr.org, test.prov.nsidr.org, test.demo.nsidr.org, test.monitoring.nsidr.org to the corresponding public IP addresses in AWS
5. Run ```vagrant provision test_cordra_nsidr_server``` to execute the ansible script which installs the software.


### Finally
Check that all the service are up and running correctly. To see the list of the services running in each machine have a look at
docs/ECOIS_subcomponents_deployment_diagram.pdf


## 4. Updating Handle records
Once the cordra instance inside the cordra_nsidr_server is running and its initial configuration and objects have been created
through the ansible script, we should update the handle records when the service is set up using a domain (eg: nsidr.org)
To do so, log in as "admin" and go to Admin->Handle Records and there click in the button Update All Handles

## 5. Digitise some Digital Specimen from DWC-A
 We can use the java project openDS_CRUD_operator https://github.com/DiSSCo/openDS_CRUD_operator to digitise specimen describe in dwc-a files
 obtained from gbif like https://www.dropbox.com/s/36ni250j6iryf0x/0034622-190918142434337_Pygmaepterys_pointieri.zip?dl=0

 After running the digitiser, we should not only have digital specimen in the codra_nsidr instance but also provenance records
 in the cordra_prov instance. As well as if codra_nsidr was set with a domain name, being able to resolve the digital specimens
 with http://hdl.handle.net/

## 6. Adding new CORDRA instances.
If we want to add a new CORDRA instance, for example for CDIDR, we need to do the following:
- Edit ```Vagrantfile``` to add configuration for the new VM
- Edit ```ansible\inventory.ini``` to add new line for the new server under the section called cordra_servers
- Edit ```ansible\group_vars\all.yml``` to add information of new Cordra instance (handle_prefix, db_name, index_name, etc )
- Edit ```ansible\roles\basic\main.yml``` to include in the task "Copying certificates" the certificate of the new server
- Edit ```ansible\roles\prometheus\templates\prometheus\prometheus.yml.j2``` to add as new target for node and blackbox exporters the new VM
- Edit ```ansible\roles\logstash\templates\logstash\cordra_pipeline.conf.j2``` to include the certificate of the new VM in the input.beats.ssl_certificate_authorities
- Edit ```ansible\roles\mongodb\templates\mongodb\initial_setup.js.j2``` to create the db for the new Cordra instance  
- Create new folder ```ansible\roles\cordra\files\NEW_CORDRA_INSTANCE``` and set the handle keys (cordra_publickey and cordra_privatekey)
- Create new folder ```ansible\roles\cordra\templates\NEW_CORDRA_INSTANCE``` and set repoInit.json.j2 and the initial_data.json.j2
- Under ```ansible\roles\grafana\templates\datasources``` create datasources for connecting to Cordra_Elasticsearch index and Cordra logstash index
- Under ```ansible\roles\grafana\templates\dashboards``` create new dashboard for showing object information of new Cordra instance    

### Funding
This code was created to deploy the dsviewer demonstrator onto a Virtual Machine, as part of the ICEDIG project
https://icedig.eu/ ICEDIG a DiSSCo Project H2020-INFRADEV-2016-2017 – Grant Agreement No. 777483 Funded by the Horizon
2020 Framework Programme of the European Union
