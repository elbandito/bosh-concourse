

## Clusters with BOSH 
(taken from concourse website https://concourse.ci/clusters-with-bosh.html)

Deploying Concourse with BOSH provides a scalable cluster with health management and rolling upgrades.

If you're not yet familiar with BOSH, learning it will be a bit of an investment, but it should pay off in spades. There are a lot of parallels between the philosophy of BOSH and Concourse.

### Prepping the Environment

To go from nothing to a BOSH managed Concourse, you'll need to do the following:

* Initialize a BOSH director, which is the server that orchestrates BOSH deployments.
* Set up your Cloud Config, which describes the infrastructure to which Concourse will be deployed.
* Upload the stemcell, which contains the base image used for VMs managed by BOSH.

You can skip some of this if you already have a working BOSH director running BOSH v255.4 and up.

### Deploying Concourse

Once you've got all that set up, download the releases listed for your version of Concourse from the Downloads section, and upload them to the BOSH director.

Next you'll need a Concourse BOSH deployment manifest. An example manifest is below; you'll want to replace the REPLACE_ME bits with whatever values are appropriate.

Note that the VM types, VM extensions, persistent disk type, and network names must come from your Cloud Config. Consult Prepping the Environment if you haven't set it up yet. You can retrieve your Cloud Config by running bosh cloud-config.

``` yaml
---
name: concourse

releases:
- name: concourse
  version: latest
- name: garden-runc
  version: latest

stemcells:
- alias: trusty
  os: ubuntu-trusty
  version: latest

instance_groups:
- name: web
  instances: 1
  # replace with a VM type from your BOSH Director's cloud config
  vm_type: REPLACE_ME
  vm_extensions:
  # replace with a VM extension from your BOSH Director's cloud config that will attach
  # this instance group to your ELB
  - REPLACE_ME
  stemcell: trusty
  azs: [z1]
  networks: [{name: private}]
  jobs:
  - name: atc
    release: concourse
    properties:
      # replace with your CI's externally reachable URL, e.g. https://ci.foo.com
      external_url: REPLACE_ME

      # replace with username/password, or configure GitHub auth
      basic_auth_username: REPLACE_ME
      basic_auth_password: REPLACE_ME

      # replace with your SSL cert and key
      tls_cert: REPLACE_ME
      tls_key: REPLACE_ME

      postgresql_database: &atc_db atc
  - name: tsa
    release: concourse
    properties: {}

- name: db
  instances: 1
  # replace with a VM type from your BOSH Director's cloud config
  vm_type: REPLACE_ME
  stemcell: trusty
  # replace with a disk type from your BOSH Director's cloud config
  persistent_disk_type: REPLACE_ME
  azs: [z1]
  networks: [{name: private}]
  jobs:
  - name: postgresql
    release: concourse
    properties:
      databases:
      - name: *atc_db
        # make up a role and password
        role: REPLACE_ME
        password: REPLACE_ME

- name: worker
  instances: 1
  # replace with a VM type from your BOSH Director's cloud config
  vm_type: REPLACE_ME
  vm_extensions:
  # replace with a VM extension from your BOSH Director's cloud config that will attach
  # sufficient ephemeral storage to VMs in this instance group.
  - REPLACE_ME
  stemcell: trusty
  azs: [z1]
  networks: [{name: private}]
  jobs:
  - name: groundcrew
    release: concourse
    properties: {}
  - name: baggageclaim
    release: concourse
    properties: {}
  - name: garden
    release: garden-runc
    properties:
      garden:
        listen_network: tcp
        listen_address: 0.0.0.0:7777

update:
  canaries: 1
  max_in_flight: 1
  serial: false
  canary_watch_time: 1000-60000
  update_watch_time: 1000-60000
You may also want to consult the property descriptions for your version of Concourse at bosh.io to see what properties you can/should tweak.

```

You may also want to consult Configuring Auth for configuring something other than basic auth for the main team.

Once you've got a manifest, just deploy it!

### Reaching the web UI

This really depends on your infrastructure. If you're deploying to AWS you may want to configure the web VM type to register with an ELB, mapping port 80 to 8080, 443 to 4443 (if you've configured TLS), and 2222 to 2222.

Otherwise you may want to configure static_ips for the web instance group and just reach the web UI directly.

### Upgrading & maintaining Concourse

With BOSH, the deployment manifest is the source of truth. This is very similar to Concourse's own philosophy, where all pipeline configuration is defined in a single declarative document.

So, to add more workers or web nodes, just change the instances value for the instance group and re-run bosh deploy.

To upgrade, just upload the new releases and re-run bosh deploy.

### Supporting external workers

If you need workers that run outside of your BOSH managed deployment (e.g. for testing with iOS or in some special network), you'll need to make some tweaks to the default configuration of the tsa job.

The TSA is the entryway for workers to join the cluster. For every new worker key pair, the TSA will be told to authorize its public key, and the workers must also know the TSA's public key ahead of time, so they know who they're connecting to.

#### Configuring the TSA's host key

First you'll need to remove the "magic" from the default deployment. By default, Concourse generates a key pair for the TSA, and gives the public key to the workers so they can trust the connection. This key occasionally gets cycled, which is fine so long as things are in one deployment, but once you have external workers you would have to update them all manually, which is annoying.

To fix this, generate a passwordless key pair via ssh-keygen, and provide the following properties in your BOSH deployment manifest:

host_key on the tsa job
the contents of the private key for the TSA server

host_public_key on the tsa job
the public key of the TSA, for the workers to use to verify the connection

For example (note that this manifest omits a bunch of stuff):

``` yml
instance_groups:
- name: web
  jobs:
  - name: tsa
    release: concourse
    properties:
      host_key: |
        -----BEGIN RSA PRIVATE KEY-----
        <a bunch of stuff>
        -----END RSA PRIVATE KEY-----
      host_public_key: "ssh-rsa blahblahblah"
  - # ...

```

After setting these properties, be sure to run bosh deploy.

#### Authorizing worker keys

We've left the worker keys auto-generated so far, which is fine for workers deployed alongside the TSA, as it'll also automatically authorize them.

External workers however will need their own private keys, and so the TSA must be told to authorize them.

To do so, set the following properties:

authorized_keys on the tsa job
the array of public keys to authorize

tsa.private_key on the groundcrew job
the private key for the worker to use when accessing the TSA

tsa.host and tsa.host_public_key on the groundcrew job
if the worker is in a separate deployment, these must be configured to reach the TSA

garden.forward_address on the groundcrew job
if the worker is in a separate deployment, this must be the locally-reachable Garden address to forward through the TSA; e.g.: 127.0.0.1:7777

`baggageclaim.forward_address` on the `groundcrew` job
if the worker is in a separate deployment, this must be the locally-reachable Baggageclaim address to forward through the TSA; e.g.: 127.0.0.1:7788

Once again, after setting these properties run `bosh deploy` to make the changes take place.

#### Making the TSA reachable

Typically the TSA and ATC will both be colocated in the same instance group. This way a single load balancer can be used with the following scheme:

* expose port `443` to `8080` (ATC's HTTP port) via SSL
* expose port `2222` to `2222` (TSA's SSH port) via TCP

Be sure to update any relevant security group rules (or equivalent in non-AWS environments) to permit both access from the outside world to port `2222` on your load balancer, and access from the load balancer to port 2222 on your TSA + ATC instances.

The BOSH deployment manifest would then colocate both jobs together, like so:

```
instance_groups:
- name: web
  vm_type: web_lb
  jobs:
  - name: atc
    release: concourse
    # ...
  - name: tsa
    release: concourse
    # ...
```

In AWS, the `web_lb` VM type would then configure `cloud_properties.elbs` to auto-register instances of `web` with an ELB. See the AWS CPI docs for more information.