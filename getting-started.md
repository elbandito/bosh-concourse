## Getting Started with BOSH CLIv2

### Links
 * [BOSH Getting Started](https://github.com/cloudfoundry/bosh-deployment)
 * [BOSH External IP](http://bosh.io/docs/init-external-ip.html)
 * [BOSH CLIv2](https://bosh.io/docs/cli-v2.html)
 * [BOSH Cloud Config](http://bosh.io/docs/cloud-config.html)
 
 Steps
 
 1. Clone `bosh-deployment` project
 ```
 git clone https://github.com/cloudfoundry/bosh-deployment.git
 ```
 
 3. Create BOSH Director for GCP Without Jump Box
 
 With the usage of a Jump Box, you'll need to expose your bosh director via `external_ip` and providing an `opsfile` to
 override default settings to use an `external_ip`.  Though you don't need to use a jump box, it's recommended for 
 production environments.  Having a jump box hides/insulates the bosh director from external traffic, making it more 
 secure or at least easier to securely manage.  While the jump box is still externally exposed, you only have to manage 
 SSH port access making the firewall rules easier to manage and less error prone.  
 
 ```
 bosh create-env ~/workspace/bosh-deployment/bosh.yml \
     --state=state.json \
     --vars-store=creds.yml \
     -o ~/workspace/bosh-deployment/gcp/cpi.yml \
     -o ~/workspace/bosh-deployment/external-ip-not-recommended.yml \
     -v external_ip=35.196.130.151 \
     -v director_name=bosh-dir \
     -v internal_cidr=10.0.0.0/24 \
     -v internal_gw=10.0.0.1 \
     -v internal_ip=10.0.0.6 \
     --var-file gcp_credentials_json=poot-92a22218d0ef.json \
     -v project_id=poot-181815 \
     -v zone=us-east1-c \
     -v tags=[bosh-internal] \
     -v network=bosh-vpc \
     -v subnetwork=bosh-subnet``
 ```
 
 3.2 Common Issues
 
 During `bosh create-env` you might encounter some issues.  Here are some of the common issues encountered while deploying
 to `GCP`.
 
 * Configure Basic IaaS
 ```
 Deploying:
   Creating instance 'bosh/0':
     Creating VM:
       Creating vm with stemcell cid 'stemcell-06b95ae4-1e11-4d3d-5d67-995d6975d54a':
         CPI 'create_vm' method responded with error: CmdError{"type":"Bosh::Clouds::CloudError","message":"Creating vm: Failed to find Google Machine Type 'n1-standard-1' in zone 'us-east1': googleapi: Error 400: Invalid value for field 'zone': 'us-east1'. Unknown zone., invalid","ok_to_retry":false}
 ```
 
 * Configure VPC and Subnet
 ```
 Deploying:
   Creating instance 'bosh/0':
     Creating VM:
       Creating vm with stemcell cid 'stemcell-06b95ae4-1e11-4d3d-5d67-995d6975d54a':
         CPI 'create_vm' method responded with error: CmdError{"type":"Bosh::Clouds::VMCreationFailed","message":"VM failed to create: googleapi: Error 400: Invalid value for field 'resource.networkInterfaces[0].networkIP': '10.0.0.6'. Requested internal IP is outside the subnetwork CIDR range., invalid","ok_to_retry":true}
 ```
 
 * Configure Firewall Settings Due to Timeout
 ```
 Deploying:
   Creating instance 'bosh/0':
     Waiting until instance is ready:
       Post https://mbus:<redacted>@10.0.0.6:6868/agent: dial tcp 10.0.0.6:6868: i/o timeout
 ```
 
 4. Create alias and login BOSH director (NEEDED TO AUTHENTICATE WITH BOSH)
 ```
 $ bosh alias-env vbox -e 192.168.50.6 --ca-cert <(bosh int ./creds.yml --path /director_ssl/ca)
 $ export BOSH_CLIENT=admin
 $ export BOSH_CLIENT_SECRET=`bosh int ./creds.yml --path /admin_password`
 ```
 
 5. Set BOSH director URL (optional).  This only saves you from having to re-type `-e ` whenever you run the `bosh` CLI.
 ```
 export BOSH_ENVIRONMENT=<target_environment>
 ```
 
 6. Setup Cloud Config
 ```
 bosh -e <target_environment> update-cloud-config cloud.yml   
 ```
 
 7. SSL Cert and Key Generation
 https://www.akadia.com/services/ssh_test_certificate.html
 
 ## Concourse
 
 ### Links
 
 [Concourse local deployment with docker-compose](http://concourse.ci/docker-repository.html)
 [concourse deployment example](https://github.com/concourse/concourse-deployment)
 [Install postgresql](https://www.codementor.io/devops/tutorial/getting-started-postgresql-server-mac-osx)
 
 * Install `garden-runc`
 >> Task 10 | 04:02:20 | Error: Release 'garden-runc' doesn't exist
 
 I had to download both the `concourse` and `garden-runc` releases and upload them via `bosh upload-release <release>`
 [here](https://concourse.ci/downloads.html).