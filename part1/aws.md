# Deploying on AWS

So you want to deploy Cloud Foundry to good old Amazon Web Services eh?  Good on you!  Well a little prep goes a long way.

## AWS Checklist

To run the `terraform` commands in this repo or do work on AWS, other than an AWS account (obviously), you'll need the following:

1. Your AWS Access Key ID
2. Your AWS Secret Key ID
3. A Name for your VPC
4. An EC2 Key Pair

### Generate Access Key

The first thing you're going to need is a combination **Access Key ID** / **Secret Key ID**.  These are generated (for IAM users) via the IAM dashboard.  If you aren't using IAM for this, you really should.

On the AWS web console, access the IAM service, and click on `Users` in the sidebar.  Then, find the user you want to do your deployment / configuration under, and click on the username.

This should bring up a summary of the user with things like the _User ARN_, _Groups_, etc.  In the bottom half of the Summary pane, you should see some tabs, and one of those tabs should be _Security Credentials_.  Click on that one.

You are strongly encouraged to generate a new Access Key, using the big blue button, for each VPC you deploy, even if you use the same IAM user for all of them.

**Make sure you save the secret key somewhere safe**, like 1Password or a Vault instance.  Amazon will be unable to give you the **Secret Key ID** if you misplace it -- your only recourse at that point is to generate a new set of keys and start over.

Now assign the **PowerUserAccess** role to your user. This user will be able to do any operation except IAM operations.  You can do this by clicking on the _Permissions_ tab and then clicking on the _attach policy_ button.

### Name Your VPC

This step is really simple -- just make one up.  The VPC name will be used to prefix all of the Network ACLs, Subnets and Security Groups, so that you can have multiple VPCs under one account without going cross-eyed trying to keep them separate.

### Generate EC2 Key Pair

The Access Key / Secret Key is used to get access to the Amazon Web Services themselves.  In order to properly deploy the NAT and Bastion Host instances to EC2, you're going to need an EC2 Key Pair.  This is the key pair you're going to need to use to SSH into the instances.

Starting from the main Amazon Web Console, go to Service > EC2, and then click the _Key Pairs_ link under _Network & Security_. The big blue `Create Key Pair` button.  Make a note of the name you chose for the key pair, because we're going to need that for our Terraform configuration.

**N.B.**: Make sure you are in the correct region (top-right corner of the black menu bar) when you create your EC2 Key Pair. Otherwise, it just plain won't work.

## Create AWS Resources with Terraform

Once the requirements for AWS are met, we can put it all together and build out your shiny new Virtual Private Cloud (VPC) in Amazon.  For this step, you're going to want to be in the `terraform/aws` sub-directory of this repository.  This Terraform configuration directly matches the [Network Plan][netplan] for the demo environment.  For deploying in production, you may need to tweak or rewrite.

Start with the following `aws.tfvars` file (substituting your actual values, of course):

```
aws_access_key = "..."
aws_secret_key = "..."
aws_vpc_name = "my-new-vpc"
aws_key_name = "bosh-ec2-key"
```

If you need to change the region or subnet, you can override the defaults by adding:

```
aws_region = "us-east-1"
network = "10.42"
```

NOTE: We recommend [a region with three availability zones][az] for production level environments.

As a quick pre-flight check, run `make manifest` to compile your Terraform plan and suss out any issues with naming, missing variables, configuration, etc.:

```
$ make manifest
terraform get -update
terraform plan -var-file aws.tfvars -out aws.tfplan
Refreshing Terraform state prior to plan...

<snip>

Plan: 33 to add, 0 to change, 0 to destroy.
```

If everything worked out you should se a summary of the plan.  If this is the first time you've done this, all of your changes should be additions.  The numbers may differ from the above output, and that's okay.

Now, to pull the trigger, run `make deploy`:

```
$ make deploy
```

Terraform will connect to AWS, using your **Access Key ID** and **Secret Key ID**, and spin up all the things it needs.  When it finishes, you should be left with a bunch of subnets, configured network ACLs, security groups, routing tables, a NAT instance (for public internet connectivity) and a Bastion host.

If the `deploy` step fails with errors like:

```
 * aws_subnet.prod-cf-edge-1: Error creating subnet: InvalidParameterValue: Value (us-east-1a) for parameter availabilityZone is invalid. Subnets can currently only be created in the following availability zones: us-east-1c, us-east-1e, us-east-1b, us-east-1d.
	status code: 400, request id: 8ddbe059-0818-48c2-a936-b551cd76cdeb
 * aws_subnet.prod-infra-1: Error creating subnet: InvalidParameterValue: Value (us-east-1a) for parameter availabilityZone is invalid. Subnets can currently only be created in the following availability zones: us-east-1c, us-east-1b, us-east-1d, us-east-1e.
	status code: 400, request id: 876f72b2-6bda-4499-98c3-502d213635eb
* aws_subnet.dev-infra-3: Error creating subnet: InvalidParameterValue: Value (us-east-1a) for parameter availabilityZone is invalid. Subnets can currently only be created in the following availability zones: us-east-1c, us-east-1b, us-east-1d, us-east-1e.
	status code: 400, request id: 66fafa81-7718-46eb-a606-e4b98e3267b9
```

you should run `make destroy` to clean up, then add a line like `aws_az1 = "d"` to replace the restricted zone.

## Prepare Bastion Host

The bastion host is an access point virtual machine that your IaaS instrumentation layer (probably Terraform) should have provisioned for you.  As such, you probably will need to consult with your IaaS provider to figure out what IP address the bastion host can be accessed at.  For example, on AWS, find the `bastion` EC2 instance and note its Elastic IP address.

You're going to need to SSH into the bastion host (as the `ubuntu` user), and unfortunately, that is also provider-specific.  In AWS, you'll just SSH to the Elastic IP, using the private half of the EC2 keypair you generated.  Other IaaS's may have other
requirements.

### Verify Keypair

You can check your SSH keypair by comparing the Amazon fingerprints.  

On the Web UI, you can check the uploaded key on the [key page][amazon-keys].

If you prefer the Amazon CLI, you can run (replacing bosh with your key name):

```
$ aws ec2 describe-key-pairs --region us-east-1 --key-name bosh|JSON.sh -b| grep 'KeyFingerprint'|awk '{ print $2 }' -
"05:ad:67:04:2a:62:e3:fb:e6:0a:61:fb:13:c7:6e:1b"
$
```

You check your private key you are using with:

```
$ openssl pkey -in ~/.ssh/bosh.pem -pubout -outform DER | openssl md5 -c
(stdin)= 05:ad:67:04:2a:62:e3:fb:e6:0a:61:fb:13:c7:6e:1b
$
```

(on OS X you need to `brew install openssl` to get OpenSSL 1.0.x and use that version)

### Setup Jumpbox

Once on the bastion host, you'll want to use the `jumpbox` script, which you can get off of Github, like so:

```
sudo curl -o /usr/local/bin/jumpbox \
https://raw.githubusercontent.com/starkandwayne/jumpbox/master/bin/jumpbox
sudo chmod 0755 /usr/local/bin/jumpbox
```

Script in hand, you can go ahead and prepare the system with
globally available utilities:

```
$ sudo jumpbox system
```

That should install some useful utilities like `jq`, `spruce`, `safe`, and `genesis` all of which will be important when we start using the bastion host to do deployments.

Next up, you're going to want to provision some normal user accounts on the bastion host, so that operations staff can login via named accounts:

```
$ jumpbox useradd
Full name: Joe User
Username:  juser
sudo password for ubuntu:
You should run `jumpbox user` now, as juser:
  sudo -iu juser
  jumpbox user

$ sudo -iu juser
$ jumpbox user
<snip>
$ mkdir ~/.ssh
$ vim ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/authorized_keys
$ logout
```

Using named accounts provides auditing (via the `sudo` logs), isolation (people won't step on each others toes on the filesystem) and customization (everyone gets to set their own prompt / shell / $EDITOR / etc.)

Once you're done setting up your users, you should log in (via SSH) as your personal account and make sure everything is working.

You can verify what's currently installed on the bastion via:

```
$ jumpbox
```

For more information, check out [the jumpbox repo][jumpbox] on Github.

Note: try not to confuse the `jumpbox` script with the jumpbox _BOSH release_.  The latter provisions the jumpbox machine as part of the deployment, provides requisite packages, and creates user accounts.  The former is really only useful for setting up / updating the bastion host.

## A Land Before Time

So you've tamed the IaaS and outfitted your bastion host with the necessary tools to deploy stuff.  First up, we have to deploy a BOSH director, which we will call proto-BOSH.

Proto-BOSH is a little different from all of the other BOSH directors we're going to deploy.  For starters, it gets deployed via `bosh-init`, whereas our environment-specific BOSH directors are going to be deployed via the proto-BOSH (and the `bosh` CLI).  It is also the only deployment that gets deployed without the benefit of a pre-existing Vault in which to store secret credentials (but, as you'll see, we're going to cheat a bit on that front).

### Proto-Vault

BOSH has secrets.  Lots of them.  Components like NATS and the database rely on secure passwords for inter-component interaction.  Ideally, we'd have a spinning Vault for storing our credentials, so that we don't have them on-disk or in a git
repository somewhere.

However, we are starting from almost nothing, so we don't have the luxury of using a BOSH-deployed Vault.  What we can do, however, is spin a single-threaded Vault server instance _on the bastion host_, and then migrate the credentials to the real Vault later.

The `jumpbox` script that we ran as part of setting up the bastion host installs the `vault` command-line utility, which includes not only the client for interacting with Vault, but also the Vault server daemon itself.

```
$ vault server -dev
==> WARNING: Dev mode is enabled!

In this mode, Vault is completely in-memory and unsealed.
Vault is configured to only have a single unseal key. The root
token has already been authenticated with the CLI, so you can
immediately begin using the Vault CLI.

The only step you need to take is to set the following
environment variables:

    export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are reproduced below in case you
want to seal/unseal the Vault or play with authentication.

Unseal Key:
781d77046dcbcf77d1423623550d28f152d9b419e09df0c66b553e1239843d89
Root Token: c888c5cd-bedd-d0e6-ae68-5bd2debee3b7

==> Vault server configuration:

         Log Level: info
             Mlock: supported: true, enabled: false
           Backend: inmem
        Listener 1: tcp (addr: "127.0.0.1:8200", tls: "disabled")
           Version: Vault v0.5.0

==> Vault server started! Log data will stream in below:

2016/06/28 13:24:54 [INFO] core: security barrier initialized (shares: 1, threshold 1)
2016/06/28 13:24:54 [INFO] core: post-unseal setup starting
2016/06/28 13:24:54 [INFO] core: mounted backend of type generic at secret/
2016/06/28 13:24:54 [INFO] core: mounted backend of type cubbyhole at cubbyhole/
2016/06/28 13:24:54 [INFO] core: mounted backend of type system at sys/
2016/06/28 13:24:54 [INFO] core: post-unseal setup complete
2016/06/28 13:24:54 [INFO] core: root token generated
2016/06/28 13:24:54 [INFO] core: pre-seal teardown starting
2016/06/28 13:24:54 [INFO] rollback: starting rollback manager
2016/06/28 13:24:54 [INFO] rollback: stopping rollback manager
2016/06/28 13:24:54 [INFO] core: pre-seal teardown complete
2016/06/28 13:24:54 [INFO] core: vault is unsealed
2016/06/28 13:24:54 [INFO] core: post-unseal setup starting
2016/06/28 13:24:54 [INFO] core: mounted backend of type generic at secret/
2016/06/28 13:24:54 [INFO] core: mounted backend of type cubbyhole at cubbyhole/
2016/06/28 13:24:54 [INFO] core: mounted backend of type system at sys/
2016/06/28 13:24:54 [INFO] core: post-unseal setup complete
2016/06/28 13:24:54 [INFO] rollback: starting rollback manager
...
```

(Note: you probably want to run this in a `tmux` session, in the
foreground.  Running it in the background sounds like a fine idea,
except that Vault is pretty chatty, and we can't redirect the
output to `/dev/null` because we need to see that root token.)

With our proto-Vault up and spinning, we can target it:

```
$ safe target proto http://127.0.0.1:8200
Now targeting proto at http://127.0.0.1:8200

$ safe targets

 proto  http://127.0.0.1:8200

$ safe auth token
Authenticating against proto at http://127.0.0.1:8200
Token: <paste your token here>

$ safe set secret/handshake knock=knock
knock: knock

$ safe read secret/handshake
--- # secret/handshake
knock: knock

```

All set!  Now we can deploy our proto-BOSH.

### Proto-BOSH

First, you're going to need a place on the bastion host to store
your deployments:

```
$ mkdir -p ~/ops
$ cd ~/ops
```

Genesis has a template for BOSH deployments (including support for
the proto-BOSH), so let's use that.

```
$ genesis new deployment --template bosh
$ cd bosh-deployments
```

Next, we'll create a site and an environment from which to deploy
our proto-BOSH.  The BOSH template comes with some site templates
to help you get started quickly, including:

- `aws` for Amazon Web Services VPC deployments
- `vsphere` for VMWare ESXi virtualization clusters
- `openstack` for OpenStack tenant deployments

We're going to use `aws`:

```
$ genesis new site --template aws aws
Created site aws (from template aws):
~/ops/docs/bosh-deployments/aws
├── README
└── site
    ├── README
    ├── disk-pools.yml
    ├── jobs.yml
    ├── networks.yml
    ├── properties.yml
    ├── releases
    ├── resource-pools.yml
    ├── stemcell
    │   ├── name
    │   ├── sha1
    │   ├── url
    │   └── version
    └── update.yml

2 directories, 13 files
```

Finally, let's create our new environment, and name it `proto`
(that's `aws/proto`, formally speaking).

```
$ genesis new environment --type bosh-init aws proto
Running env setup hook: ~/ops/bosh-deployments/.env_hooks/setup

 proto  http://127.0.0.1:8200

Use this Vault for storing deployment credentials?  [yes or no]
yes
Setting up credentials in vault, under secret/aws/proto/bosh
.
└── secret/aws/proto/bosh
    ├── blobstore/
    │   ├── agent
    │   └── director
    ├── db
    ├── nats
    ├── users/
    │   ├── admin
    │   └── hm
    └── vcap


Created environment aws/proto:
~/ops/bosh-deployments/aws/proto
├── cloudfoundry.yml
├── credentials.yml
├── director.yml
├── Makefile
├── monitoring.yml
├── name.yml
├── networking.yml
├── properties.yml
├── README
└── scaling.yml

0 directories, 10 files
```

(Note: don't forget that `--type bosh-init` flag, it's very
important. otherwise, you'll run into problems with your
deployment)

The template helpfully generated all new credentials for us and
stored them in our proto-Vault, under the `secret/aws/proto/bosh`
subtree.  Later, we'll migrate this subtree over to our real
Vault, once it is up and spinning.

Let's head into the `proto/` environment directory and see if we
can create a manifest, or (a more likely case) we still have to
provide some critical information:

```
$ cd aws/proto
$ make manifest
9 error(s) detected:
 - $.meta.aws.access_key: Please supply an AWS Access Key
 - $.meta.aws.azs.z1: What Availability Zone will BOSH be in?
 - $.meta.aws.region: What AWS region are you going to use?
 - $.meta.aws.secret_key: Please supply an AWS Secret Key
 - $.meta.aws.ssh_key_name: What is your full key name?
 - $.meta.aws.default_sgs: What Security Groups?
 - $.meta.aws.private_key: What is the local path to the Amazon Private Key for this deployment?
 - $.networks.default.subnets: Specify subnets for your BOSH vm's network
 - $.meta.shield_public_key: Specify the SSH public key from this environment's SHIELD daemon
Availability Zone will BOSH be in?


Failed to merge templates; bailing...
Makefile:22: recipe for target 'manifest' failed
make: *** [manifest] Error 5
```

Drat.  Luckily, a lot of these are duplicates, most likely from a
`(( grab ... ))` operation.  Let's focus on the `$.meta` subtree,
since that's where most parameters are defined in Genesis
templates:

```
- $.meta.aws.access_key: Please supply an AWS Access Key
- $.meta.aws.azs.z1: What Availability Zone will BOSH be in?
- $.meta.aws.region: What AWS region are you going to use?
- $.meta.aws.secret_key: Please supply an AWS Secret Key
```

(As you can see, I'm using the AWS site for this run-through.  On
other environments, using other templates, your mileage will
vary.)

This is easy enough to supply.  We'll put these properties in
`properties.yml`:

```
$ cat properties.yml
---
meta:
  aws:
    region: us-west-2
    azs:
      z1: (( concat meta.aws.region "a" ))
    access_key: (( vault "secret/aws:access_key" ))
    secret_key: (( vault "secret/aws:secret_key" ))
```

I use the `(( concat ... ))` operator to [DRY][DRY] up the
configuration.  This way, if we need to move the BOSH director to
a different region (for whatever reason) we just change
`meta.aws.region` and the availability zone just tacks on "a".

(We use the "a" availability zone because that's where our subnet
is located.)

I also configured the AWS access and secret keys by pointing
Genesis to the Vault.  Let's go put those credentials in the
Vault:

```
$ safe set secret/aws access_key secret_key
access_key [hidden]:
access_key [confirm]:

secret_key [hidden]:
secret_key [confirm]:

```

Let's try that `make manifest` again.

```
$ make manifest`
7 error(s) detected:
 - $.meta.aws.ssh_key_name: What is your full key name?
 - $.meta.aws.default_sgs: What Security Groups?
 - $.meta.aws.private_key: What is the local path to the Amazon Private Key for this deployment?
 - $.networks.default.subnets: Specify subnets for your BOSH vm's network
 - $.properties.aws.default_key_name: What is your full key name?
 - $.properties.aws.default_security_groups: What Security Groups?
 - $.meta.shield_public_key: Specify the SSH public key from this environment's SHIELD daemon


Failed to merge templates; bailing...
Makefile:22: recipe for target 'manifest' failed
make: *** [manifest] Error 5
```

Better.  Note that we still have some `(( grab ... ))` calls in
there, leading to the duplication.

> Once [issue #129][spruce-129] is fixed, the duplication of
> `(( param ... ))` violations should go away, leading to cleaner
> error messages and a smoother setup process.

Let's configure our `cloud_provider` for AWS, using our EC2
keypair.

```
$ cat properties.yml
---
meta:
  aws:
    region: us-west-2
    azs:
      z1: (( concat meta.aws.region "a" ))
    access_key: (( vault "secret/aws:access_key" ))
    secret_key: (( vault "secret/aws:secret_key" ))
    private_key: /path/to/the/ec2/key.pem
    ssh_key_name: your-ec2-keypair-name
    default_sgs:
      - restricted
```

Once more, with feeling:

```
$ make manifest
2 error(s) detected:
 - $.networks.default.subnets: Specify subnets for your BOSH vm's network
 - $.meta.shield_public_key: Specify the SSH public key from this environment's SHIELD daemon


Failed to merge templates; bailing...
Makefile:22: recipe for target 'manifest' failed
make: *** [manifest] Error 5
```

Excellent.  We're down to two issues.

We haven't deployed a SHIELD yet, so it may seem a bit odd that
we're being asked for an SSH public key.  When we deploy our
proto-BOSH via `bosh-init`, we're going to spend a fair chunk of
time compiling packages on the bastion host before we can actually
create and update the director VM.  `bosh-init` will delete the
director VM before it starts this compilation phase, so we will be
unable to do _anything_ while `bosh-init` is hard at work.  The
whole process takes about 30 minutes, so we want to minimize the
number of times we have to re-deploy proto-BOSH.  By specifying
the SHIELD agent configuration up-front, we skip a re-deploy after
SHIELD itself is up.

Let's leverage our Vault to create the SSH keypair for BOSH.
`safe` has a handy builtin for doing this:

```
$ safe ssh secret/aws/proto/shield/keys/core
$ safe get secret/aws/proto/shield/keys/core
--- # secret/aws/proto/shield/keys/core
fingerprint: 40:9b:11:82:67:41:23:a8:c2:87:98:5d:ec:65:1d:30
private: |
  -----BEGIN RSA PRIVATE KEY-----
  MIIEowIBAAKCAQEA+hXpB5lmNgzn4Oaus8nHmyUWUmQFmyF2pa1++2WBINTIraF9
  ... etc ...
  5lm7mGwOCUP8F1cdPmpPNCkoQ/dx3T5mnsCGsb3a7FVBDDBje1hs
  -----END RSA PRIVATE KEY-----
public: |
  ssh-rsa AAAAB3NzaC...4vbnncAYZPTl4KOr
```

(output snipped for brevity and security; but mostly brevity)

Now we can put references to our Vaultified keypair in
`credentials.yml`:

```
$ cat credentials.yml
---
meta:
  shield_public_key: (( vault "secret/aws/proto/shield/keys/core:public" ))
```

You may want to take this opportunity to migrate
credentials-oriented keys from `properties.yml` into this file.

Now, we should have only a single error left when we `make
manifest`:

```
$ make manifest
1 error(s) detected:
 - $.networks.default.subnets: Specify subnets for your BOSH vm's network


Failed to merge templates; bailing...
Makefile:22: recipe for target 'manifest' failed
make: *** [manifest] Error 5
```

So it's down to networking.

Refer back to your [Network Plan][netplan], and find the
subnet for the proto-BOSH.  If you're using the plan in this
repository, that would be `10.4.1.0/24`, and we're allocating
`10.4.1.0/28` to our BOSH director.  Our `networking.yml` file,
then, should look like this:

```
$ cat networking.yml
---
networks:
  - name: default
    subnets:
      - range:    10.4.1.0/24
        gateway:  10.4.1.1
        dns:     [10.4.0.2]
        cloud_properties:
          subnet: subnet-xxxxxxxx # <-- your AWS Subnet ID
          security_groups: [wide-open]
        reserved:
          - 10.4.1.2 - 10.4.1.3    # Amazon reserves these
            # proto-BOSH is in 10.4.1.0/28
          - 10.4.1.16 - 10.4.1.254 # Allocated to other deployments
        static:
          - 10.4.1.4
```

Our range is that of the actual subnet we are in, `10.4.1.0/24`
(in reality, the `/28` allocation is merely a tool of bookkeeping
that simplifies ACLs and firewall configuration).  As such, our
Amazon-provided default gateway is 10.4.1.1 (the first available
IP) and our DNS server is 10.4.0.2.

We identify our AWS-specific configuration under
`cloud_properties`, by calling out what AWS Subnet we want the EC2
instance to be placed in, and what EC2 Security Groups it should
be subject to.

Under the `reserved` block, we reserve the IPs that Amazon
reserves for its own use (see [Amazon's
documentation][aws-subnets], specifically the "Subnet sizing"
section), and everything outside of `10.4.1.0/28` (that is,
`10.4.1.16` and above).

Finally, in `static` we reserve the first usable IP (`10.4.1.4`)
as static.  This will be assigned to our `bosh/0` director VM.

Now, `make manifest` should succeed (no output is a good sign),
and we should have a full manifest at `manifests/manifest.yml`:

```
$ make manifest
$ ls -l manifests/
total 8
-rw-r--r-- 1 ops staff 4572 Jun 28 14:24 manifest.yml
```

> TODO: I had to `echo bosh-init > .type` to engage the bosh-init
> style of deployment.  How do we want to handle that?  Does
> Genesis need an update for a `--type` flag to `new env`?

> TODO: i also had to copy the aws key up to the bastion host.


```
$ make deploy
No existing genesis-created bosh-init statefile detected. Please
help genesis find it.
Path to existing bosh-init statefile (leave blank for new
deployments):
Deployment manifest: '~/ops/bosh-deployments/aws/proto/manifests/.deploy.yml'
Deployment state: '~/ops/bosh-deployments/aws/proto/manifests/.deploy-state.json'

Started validating
  Downloading release 'bosh'... Finished (00:00:09)
  Validating release 'bosh'... Finished (00:00:03)
  Downloading release 'bosh-aws-cpi'... Finished (00:00:02)
  Validating release 'bosh-aws-cpi'... Finished (00:00:00)
  Downloading release 'shield'... Finished (00:00:10)
  Validating release 'shield'... Finished (00:00:02)
  Validating cpi release... Finished (00:00:00)
  Validating deployment manifest... Finished (00:00:00)
  Downloading stemcell... Finished (00:00:01)
  Validating stemcell... Finished (00:00:00)
Finished validating (00:00:29)
...
```

(At this point, `bosh-init` starts the tedious process of
compiling all the things.  End-to-end, this is going to take about
a half an hour, so you probably want to go play [a game][slither]
or grab a cup of tea.)

...

All done?  Verify the deployment by trying to `bosh target` the
newly-deployed Director.  First you're going to need to get the
password out of our proto-Vault.

```
$ safe get secret/aws/proto/bosh/users/admin
--- # secret/mgmt/proto/bosh/users/admin
password: super-secret
```

Then, run target the director:

```
$ bosh target https://10.4.1.4:25555 proto-bosh
Target set to `aws-proto-bosh'
Your username: admin
Enter password:
Logged in as `admin'

$ bosh status
Config
             ~/.bosh_config

Director
  Name       aws-proto-bosh
  URL        https://10.4.1.4:25555
  Version    1.3232.2.0 (00000000)
  User       admin
  UUID       a43bfe93-d916-4164-9f51-c411ee2110b2
  CPI        aws_cpi
  dns        disabled
  compiled_package_cache disabled
  snapshots  disabled

Deployment
  not set
```

All set!

Before you move onto the next step, you should commit your local
deployment files to version control, and push them up _somewhere_.
It's ok, thanks to Vault, there are no credentials or anything
sensitive in the Genesis template files.

### Vault

Now that we have a proto-BOSH director, we can use it to deploy
our real Vault.  We'll start with the Genesis template for Vault:

```
$ cd ~/ops
$ genesis new deployment --template vault
$ cd vault-deployments
```

As before (and as will become almost second-nature soon), let's
create our `aws` site using the `aws` template, and then create
the `proto` environment inside of that site.

```
$ genesis new site --template aws aws
$ genesis new environment aws proto
```

Answer yes twice and then enter a name for your Vault instance when prompted for a FQDN.

```
$ cd aws/proto
$ make manifest
10 error(s) detected:
 - $.compilation.cloud_properties.availability_zone: Define the z1 AWS availability zone
 - $.meta.aws.azs.z1: Define the z1 AWS availability zone
 - $.meta.aws.azs.z2: Define the z2 AWS availability zone
 - $.meta.aws.azs.z3: Define the z3 AWS availability zone
 - $.networks.vault_z1.subnets: Specify the z1 network for vault
 - $.networks.vault_z2.subnets: Specify the z2 network for vault
 - $.networks.vault_z3.subnets: Specify the z3 network for vault
 - $.resource_pools.small_z1.cloud_properties.availability_zone: Define the z1 AWS availability zone
 - $.resource_pools.small_z2.cloud_properties.availability_zone: Define the z2 AWS availability zone
 - $.resource_pools.small_z3.cloud_properties.availability_zone: Define the z3 AWS availability zone


Failed to merge templates; bailing...
Makefile:22: recipe for target 'manifest' failed
make: *** [manifest] Error 5
```

Vault is pretty self-contained, and doesn't have any secrets of
its own.  All you have to supply is your network configuration,
and any IaaS settings.

Referring back to our [Network Plan][netplan] again, we
find that Vault should be striped across three zone-isolated
networks:

  - **10.4.1.16/28** in zone 1 (a)
  - **10.4.2.16/28** in zone 2 (b)
  - **10.4.3.16/28** in zone 3 (c)

First, lets do our AWS-specific region/zone configuration, along with our Vault HA fully-qualified domain name:

```
$ cat properties.yml
---
meta:
  aws:
    region: us-west-2
    azs:
      z1: (( concat meta.aws.region "a" ))
      z2: (( concat meta.aws.region "b" ))
      z3: (( concat meta.aws.region "c" ))
properties:
  vault:
    ha:
      domain: 10.4.1.16
```

Our `/28` ranges are actually in their corresponding `/24` ranges
because the `/28`'s are (again) just for bookkeeping and ACL
simplification.  That leaves us with this for our
`networking.yml`:

```
$ cat networking.yml
---
networks:
  - name: vault_z1
    subnets:
      - range:    10.4.1.0/24
        gateway:  10.4.1.1
        dns:     [10.4.0.2]
        cloud_properties:
          subnet: subnet-xxxxxxxx  # <--- your AWS Subnet ID
          security_groups: [wide-open]
        reserved:
          - 10.4.1.2 - 10.4.1.3    # Amazon reserves these
          - 10.4.1.4 - 10.4.1.15   # Allocated to other deployments
            # Vault (z1) is in 10.4.1.16/28
          - 10.4.1.32 - 10.4.1.254 # Allocated to other deployments
        static:
          - 10.4.1.16 - 10.4.1.18

  - name: vault_z2
    subnets:
      - range:    10.4.2.0/24
        gateway:  10.4.2.1
        dns:     [10.4.2.2]
        cloud_properties:
          subnet: subnet-yyyyyyyy  # <--- your AWS Subnet ID
          security_groups: [wide-open]
        reserved:
          - 10.4.2.2 - 10.4.2.3    # Amazon reserves these
          - 10.4.2.4 - 10.4.2.15   # Allocated to other deployments
            # Vault (z2) is in 10.4.2.16/28
          - 10.4.2.32 - 10.4.2.254 # Allocated to other deployments
        static:
          - 10.4.2.16 - 10.4.2.18

  - name: vault_z3
    subnets:
      - range:    10.4.3.0/24
        gateway:  10.4.3.1
        dns:     [10.4.3.2]
        cloud_properties:
          subnet: subnet-zzzzzzzz  # <--- your AWS Subnet ID
          security_groups: [wide-open]
        reserved:
          - 10.4.3.2 - 10.4.3.3    # Amazon reserves these
          - 10.4.3.4 - 10.4.3.15   # Allocated to other deployments
            # Vault (z3) is in 10.4.3.16/28
          - 10.4.3.32 - 10.4.3.254 # Allocated to other deployments
        static:
          - 10.4.3.16 - 10.4.3.18
```

That's a ton of configuration, but when you break it down it's not
all that bad.  We're defining three separate networks (one for
each of the three availability zones).  Each network has a unique
AWS Subnet ID, but they share the same EC2 Security Groups, since
we want uniform access control across the board.

The most difficult part of this configuration is getting the
reserved ranges and static ranges correct, and self-consistent
with the network range / gateway / DNS settings.  This is a bit
easier since our network plan allocates a different `/24` to each
zone network, meaning that only the third octet has to change from
zone to zone (x.x.1.x for zone 1, x.x.2.x for zone 2, etc.)

Now, let's try a `make manifest` again (no output is a good sign):

```
$ make manifest
```

Before we can deploy we need to upload the stemcell:

```
$ bosh upload stemcell https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent?v=3232.8
```

With the `stemcell` in place, let's give the deploy a whirl:

```
$ make deploy
Acting as user 'admin' on 'aws-proto-bosh'
Checking whether release consul/20 already exists...NO
Using remote release `https://bosh.io/d/github.com/cloudfoundry-community/consul-boshrelease?v=20'

Director task 1

```

Thanks to Genesis, we don't even have to upload the BOSH releases
(or stemcells) ourselves!

### Initializing Your Global Vault

Now that the Vault software is spinning, you're going to need to
initialize the Vault, which generates a root token for interacting
with the Vault, and a set of 5 _seal keys_ that will be used to
unseal the Vault so that you can interact with it.

First off, we need to find the IP addresses of our Vault nodes:

```
$ bosh vms aws-proto-vault
+---------------------------------------------------+---------+-----+----------+-----------+
| VM                                                | State   | AZ  | VM Type  | IPs       |
+---------------------------------------------------+---------+-----+----------+-----------+
| vault_z1/0 (9fe19a85-e9ed-4bab-ac80-0d3034c5953c) | running | n/a | small_z1 | 10.4.1.16 |
| vault_z2/0 (13a46946-cd06-46e5-8672-89c40fd62e5f) | running | n/a | small_z2 | 10.4.2.16 |
| vault_z3/0 (3b234173-04d4-4bfb-b8bc-5966592549e9) | running | n/a | small_z3 | 10.4.3.16 |
+---------------------------------------------------+---------+-----+----------+-----------+
```

(Your UUIDs may vary, but the IPs should be close.)

Let's target the vault at 10.4.1.16:

```
$ export VAULT_ADDR=https://10.4.1.16:8200
$ export VAULT_SKIP_VERIFY=1
```

We have to set `$VAULT_SKIP_VERIFY` to a non-empty value becase we
used self-signed certificates when we deployed our Vault.
Ideally, you'll be working with real certificates, and won't have
to perform this step.

Let's initialize the Vault:

```
$ vault init
Unseal Key 1: c146f038e3e6017807d2643fa46d03dde98a2a2070d0fceaef8217c350e973bb01
Unseal Key 2: bae9c63fe2e137f41d1894d8f41a73fc768589ab1f210b1175967942e5e648bd02
Unseal Key 3: 9fd330a62f754d904014e0551ac9c4e4e520bac42297f7480c3d651ad8516da703
Unseal Key 4: 08e4416c82f935570d1ca8d1d289df93a6a1d77449289bac0fa9dc8d832c213904
Unseal Key 5: 2ddeb7f54f6d4f335010dc5c3c5a688b3504e41b749e67f57602c0d5be9b042305
Initial Root Token: e63da83f-c98a-064f-e4c0-cce3d2e77f97

Vault initialized with 5 keys and a key threshold of 3. Please
securely distribute the above keys. When the Vault is re-sealed,
restarted, or stopped, you must provide at least 3 of these keys
to unseal it again.

Vault does not store the master key. Without at least 3 keys,
your Vault will remain permanently sealed.
```

**Store these seal keys and the root token somewhere safe!!**
(A password manager like 1password is an excellent option here.)

Unlike the dev-mode proto-Vault we spun up at the very outset,
this Vault comes up sealed, and needs to be unsealed using three
of the five keys above, so let's do that.

```
$ vault unseal
Key (will be hidden):
Sealed: true
Key Shares: 5
Key Threshold: 3
Unseal Progress: 1

$ vault unseal
...

$ vault unseal
Key (will be hidden):
Sealed: false
Key Shares: 5
Key Threshold: 3
Unseal Progress: 0
```

Now, let's switch back to using `safe`:

```
$ safe target https://10.4.1.16:8200 ops
Now targeting prod at https://10.4.1.16:8200

$ safe auth token
Authenticating against ops at https://10.4.1.16:8200
Token:

$ safe set secret/handshake knock=knock
knock: knock
```

### Migrating Credentials

You should now have two `safe` targets, one for the proto-Vault
(named 'proto') and another for the real Vault (named 'ops'):

```
$ safe targets

(*) ops     https://10.4.1.16:8200
    proto   http://127.0.0.1:8200

```

Our `ops` Vault should be empty; we can verify that with `safe
tree`:

```
$ safe target ops -- tree
Now targeting ops at https://10.4.1.16:8200
.
└── secret
    └── handshake

```

`safe` sports a handy import/export feature that can be used to
move credentials securely between Vaults, without touching disk,
which is exactly what we need to migrate from our proto-Vault to
our real one:

```
$ safe target proto -- export secret | \
  safe target ops   -- import
Now targeting ops at https://10.4.1.16:8200
Now targeting proto at http://127.0.0.1:8200
wrote secret/aws/proto/shield/webui
wrote secret/aws/test/bosh/db
wrote secret/aws/test/bosh/nats
wrote secret/aws/proto/bosh/blobstore/director
wrote secret/aws/proto/shield/daemon
wrote secret/aws/proto/shield/db
wrote secret/aws/proto/shield/keys/core
wrote secret/aws/proto/shield/sessionsdb
wrote secret/aws/test/bosh/blobstore/director
wrote secret/aws/test/bosh/users/admin
wrote secret/aws/test/bosh/users/hm
wrote secret/aws/proto/bosh/blobstore/agent
wrote secret/aws/proto/bosh/users/admin
wrote secret/aws/proto/bosh/users/hm
wrote secret/aws/proto/bosh/db
wrote secret/aws/test/bosh/blobstore/agent
wrote secret/aws/test/bosh/vcap
wrote secret/handshake
wrote secret/aws/proto/bosh/nats
wrote secret/aws/proto/bosh/vcap
wrote secret/aws/proto/vault/tls

$ safe target ops -- tree
Now targeting ops at https://10.4.1.16:8200
.
└── secret
    ├── aws/
    │   ├── proto/
    │   │   ├── bosh/
    │   │   │   ├── blobstore/
    │   │   │   │   ├── agent
    │   │   │   │   └── director
    │   │   │   ├── db
    │   │   │   ├── nats
    │   │   │   ├── users/
    │   │   │   │   ├── admin
    │   │   │   │   └── hm
    │   │   │   └── vcap
    │   │   ├── shield/
    │   │   │   ├── daemon
    │   │   │   ├── db
    │   │   │   ├── keys/
    │   │   │   │   └── core
    │   │   │   ├── sessionsdb
    │   │   │   └── webui
    │   │   └── vault/
    │   │       └── tls
    │   └── test/
    │       └── bosh/
    │           ├── blobstore/
    │           │   ├── agent
    │           │   └── director
    │           ├── db
    │           ├── nats
    │           ├── users/
    │           │   ├── admin
    │           │   └── hm
    │           └── vcap
    └── handshake
```

Voila!  We now have all of our credentials in our real Vault, and
we can kill the proto-Vault server process!

```
$ sudo pkill vault
```

## Deploying Bolo Monitoring

Bolo is a monitoring system that collects metrics and state data
from your BOSH deployments, aggregates it, and provides data
visualization and notification primitives.

You may opt to deploy Bolo once for all of your environments, in
which case it belongs in your management network, or you may
decide to deploy per-environment Bolo installations.  What you
choose mostly only affects your network topology / configuration.

### Deploying Bolo

To get started, you're going to need to create a Genesis
deployments repo for your Bolo deployments:

```
$ cd ~/ops
$ genesis new deployment --template bolo
$ cd bolo-deployments
```

Next, we'll create a site for your datacenter or VPC.  The bolo
template deployment offers some site templates to make getting
things stood up quick and easy, including:

- `aws` for Amazon Web Services VPC deployments
- `vsphere` for VMWare ESXi virtualization clusters
- `bosh-lite` for deploying and testing locally

For purposes of illustration, let's choose `aws`:

```
$ genesis new site --template aws mgmt
Created site mgmt (from template aws):
~/ops/bolo-deployments/mgmt
├── README
└── site
    ├── disk-pools.yml
    ├── jobs.yml
    ├── networks.yml
    ├── properties.yml
    ├── releases
    ├── resource-pools.yml
    ├── stemcell
    │   ├── name
    │   └── version
    └── update.yml

2 directories, 10 files
```

(Note: if you are deploying per-environment Bolo installations,
you may want to choose something more environment-appropriate than
`mgmt`...)

Now, we can create our environment.  I like to call this `prod`,
in case we decide to build staging / sandbox environments for
deployment runways later.

```
$ cd mgmt/
$ genesis new environment prod
Created environment mgmt/prod:
~/ops/bolo-deployments/mgmt/prod
├── Makefile
├── README
├── cloudfoundry.yml
├── credentials.yml
├── director.yml
├── monitoring.yml
├── name.yml
├── networking.yml
├── properties.yml
└── scaling.yml

0 directories, 10 files
```

Bolo deployments have no secrets, so there isn't much in the way
of environment hooks for setting up credentials.

### Configuring Bolo For Amazon AWS

We need to configure the following things for an AWS deployment of
bolo:

- Availability Zone (via `meta.az`)
- Networking configuration

First, let's do the availability zone:

```
$ cd prod/
$ cat properties.yml
---
meta:
  az:
    us-west-2c
```

Then, open up `networking.yml` and fill out your networking
configuration.  For purposes of illustration, let's assume we're
going to deploy our Bolo into the `10.4.0.128/26` subdivision of
our `10.4.0.0/24` subnet.  For reference, here's the details on
the `/26` network:

```
-[ipv4 : 10.4.0.128/26] - 0

[CIDR]
Host address            - 10.4.0.128
Host address (decimal)  - 168034432
Host address (hex)      - A040080
Network address         - 10.4.0.128
Network mask            - 255.255.255.192
Network mask (bits)     - 26
Network mask (hex)      - FFFFFFC0
Broadcast address       - 10.4.0.191
Cisco wildcard          - 0.0.0.63
Addresses in network    - 64
Network range           - 10.4.0.128 - 10.4.0.191
Usable range            - 10.4.0.129 - 10.4.0.190
```

So that's 64 available hosts, starting at 10.4.0.128 and continuing
to 10.4.0.191.  Let's reserve the first 16 IPs for static
allocation, and let the compilation VMs / dynamically-allocated
VMs use the rest:

```
$ cat networking.yml
---
networks:
  - name: bolo
    type: manual
    subnets:
    - range: 10.4.0.0/24
      gateway: 10.4.0.1
      cloud_properties:
        subnet: subnet-XXXXXXXX # <--------- you'll want to change this
        security_groups: [sg-XXXXXXXX] # <-- also, change this
      dns: [10.4.0.2]
      reserved:
        - 10.4.0.2   - 10.4.0.127  # everything before our /26
        - 10.4.0.192 - 10.4.0.254  # everything after our /26
      static:
        - 10.4.0.128 - 10.4.0.144  # first 16 IPs

jobs:
  - name: bolo
    networks:
      - name: bolo
        static_ips: (( static_ips 0 ))
```

### Deploying

You can validate your manifest by running `make manifest` and
ensuring that you get no errors (no output is a good sign)

Then, you can deploy to your BOSH director via `make deploy`

Once you've deployed, you can validate the deployment by accessing
the Gnossis web interface on your `bolo/0` VM.  You can find the
IP via `bosh vms`, and just visit it in a browser, over HTTP
(standard port).

Out of the box, the Bolo installation will begin monitoring itself
for general host health (the `linux` collector), so you should
have graphs.

### Configuring dbolo Agents

Now that you have a Bolo installation, you're going to want to
configure your other deployments to use it.  To do that, you'll
need to add the `bolo` release to the deployment (if it isn't
already there), add the `dbolo` template to all the VMs you want
monitored, and configure `dbolo` to submit metrics to your
`bolo/0` VM in the bolo deployment.

(Note that this may require configuration of network ACLs,
security groups, etc. -- if you experience issues with this step,
you might want to start looking in those areas first)

To add the release:

```
$ cd ~/ops/shield-deployments
$ genesis add release shield latest
$ cd mgmt/prod
$ genesis use release shield
```

If you do a `make manifest` at this point, you should see a new
release being added to the top-level `releases` list.

To configure dbolo, you're going to want to add a line like the
last one here to all of your job template definitions:

```
jobs:
  - name: whatever
    templates:
      - { release: bolo, name: dbolo }
```

Then, to configure `dbolo` to submit to your Bolo installation,
add the `dbolo.submission.address` property either globally or
per-job (strong recommendation for global, by the way).  You can
do this in `properties.yml`

```
properties:
  dbolo:
    submission:
      address: 10.4.0.128
```

As before, you can get the IP address of the `bolo/0` VM by
running `bosh vms` against your BOSH director.

### Configuring Specific Monitoring

If you have specific monitoring requirements, above and beyond
the stock host-health checks that the `linux` collector provides,
you can add per-job (or global) properties like this (in
properties.yml, again):

```
jobs:
  - name: shield
    properties:
      dbolo:
        collectors:
          - { every: 20s, run: 'linux' }
          - { every: 20s, run: 'httpd' }
          - { every: 20s, run: 'process -n nginx -m nginx' }
```

(Remember that you will need to supply the `linux` collector
configuration, since Bolo skips the automatic `dbolo` settings you
get for free when you specify your own configuration.)

### Further Reading on Bolo

More information can be found in the [Bolo BOSH Release README][bolo]
which contains a wealth of information about available graphs,
collectors, and deployment properties.

## Deploying SHIELD Backups

SHIELD is our backup solution.  We use it to configure and
schedule regular backups of data systems that are important to our
running operation, like the BOSH database, Concourse, and Cloud
Foundry.

We'll start out with the Genesis template for SHIELD:

```
$ cd ~/ops
$ genesis new deployment --template shield
$ cd shield-deployments
```

Now we can set up our `aws` site using the `aws` template, with a
`proto` environment inside of it:

```
$ genesis new site --template aws aws
$ genesis new environment aws proto
$ cd aws/proto
$ make manifest
5 error(s) detected:
 - $.compilation.cloud_properties.availability_zone: What availability zone is SHIELD deployed to?
 - $.meta.az: What availability zone is SHIELD deployed to?
 - $.networks.shield.subnets: Specify your shield subnet
 - $.properties.shield.daemon.ssh_private_key: Specify the SSH private key that the daemon will use to talk to the agents
 - $.resource_pools.small.cloud_properties.availability_zone: What availability zone is SHIELD deployed to?


Failed to merge templates; bailing...
Makefile:22: recipe for target 'manifest' failed
make: *** [manifest] Error 5
```

By now, this should be old hat.  According to the [Network
Plan][netplan], the SHIELD deployment belongs in the
**10.4.1.32/28** network, in zone 1 (a).  Let's put that
information into `properties.yml`:

```
$ cat properties.yml
---
meta:
  az: us-west-2a
```

As we found with Vault, the `/28` range is actually in it's outer
`/24` range, since we're just using the `/28` subdivision for
convenience.

```
$ cat networking.yml
---
networks:
  - name: shield
    subnets:
      - range:    10.4.1.0/24
        gateway:  10.4.1.1
        dns:     [10.4.0.2]
        cloud_properties:
          subnet: subnet-xxxxxxxx  # <--- your AWS Subnet ID
          security_groups: [wide-open]
        reserved:
          - 10.4.1.2 - 10.4.1.3    # Amazon reserves these
          - 10.4.1.4 - 10.4.1.31   # Allocated to other deployments
            # SHIELD is in 10.4.1.32/28
          - 10.4.1.48 - 10.4.1.254 # Allocated to other deployments
        static:
          - 10.4.1.32 - 10.4.1.34
```

(Don't forget to change your `subnet` to match your AWS VPC
configuration.)

Finally, if you recall, we already generated an SSH keypair for
SHIELD, so that we could pre-deploy the pubic key to our
Proto-BOSH.  We stuck it in the Vault, at
`secret/aws/proto/shield/keys/core`, so let's get it back out for this
deployment:

```
$ cat credentials.yml
---
properties:
  shield:
    daemon:
      ssh_private_key: (( vault meta.vault_prefix "/keys/core:private"))
```

Now, our `make manifest` should succeed (and not complain)

```
$ make manifest
```

Time to deploy!

```
$ make deploy
Acting as user 'admin' on 'aws-proto-bosh'
Checking whether release shield/6.3.0 already exists...NO
Using remote release `https://bosh.io/d/github.com/starkandwayne/shield-boshrelease?v=6.3.0'

Director task 13
  Started downloading remote release > Downloading remote release

```

Once that's complete, you will be able to access your SHIELD
deployment, and start configuring your backup jobs.  Before we do
that, however, let's prepare our Amazon infrastructure to store
backups in S3, one of SHIELD's built-in archive storage systems.

### Setting up AWS S3 For Backup Archives

To help keep things isolated, we're going to set up a brand new
IAM user just for backup archive storage.  It's a good idea to
name this user something like `backup` or `shield-backup` so that
no one tries to re-purpose it later, and so that it doesn't get
deleted.

You're also going to want to provision a dedicated S3 bucket to
store archives in, and name it something descriptive, like
`codex-backups`.

Since the generic S3 bucket policy is a little open (and we don't
want random people reading through our backups), we're going to
want to create our own policy:

```
{
  "Statement": [
    {
      "Effect"   : "Allow",
      "Action"   : "s3:ListAllMyBuckets",
      "Resource" : "arn:aws:iam:xxxxxxxxxxxx:user/zzzzz",
    },
    {
      "Effect"   : "Allow",
      "Action"   : "s3:*",
      "Resource" : [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*",
      ]
    }
  ]
}
```



## Deploying Concourse

If we're not already targeting the ops vault, do so now to save frustration later.

```
$ safe target "https://10.4.1.16:8200" proda
Now targeting proda at https://10.4.1.16:8200
$ export VAULT_SKIP_VERIFY=1
```

From the `~/ops` folder let's generate a new `concourse` deployment, using the `--template` flag.

```
$ genesis new deployment --template concourse
$ cd concourse-deployments/
```

Inside the `global` deployment level goes the site level definition.  For this concourse setup we'll use an `aws` template for an `aws` site.

```
$ genesis new site --template aws aws
Created site aws (from template aws):
/home/user/ops/concourse-deployments/aws
├── README
└── site
    ├── disk-pools.yml
    ├── jobs.yml
    ├── networks.yml
    ├── properties.yml
    ├── releases
    ├── resource-pools.yml
    ├── stemcell
    │   ├── name
    │   └── version
    └── update.yml

2 directories, 10 files
```

Finally now, because our vault is setup and targeted correctly we can generate our `environment` level configurations.  And begin the process of setting up the specific parameters for our environment.

```
~/ops/concourse-deployments$ genesis new environment aws proto
Running env setup hook: /home/user/ops/concourse-deployments/.env_hooks/00_confirm_vault

(*) proda	https://10.42.1.16:8200
    prodb	https://10.42.2.16:8200
    prodc	https://10.42.3.16:8200
    proto	http://127.0.0.1:8200

Use this Vault for storing deployment credentials?  [yes or no] yes
Running env setup hook: /home/user/ops/concourse-deployments/.env_hooks/gen_creds
Generating credentials for Concource CI
Created environment aws/proto:
/home/user/ops/concourse-deployments/aws/proto
├── cloudfoundry.yml
├── credentials.yml
├── director.yml
├── Makefile
├── monitoring.yml
├── name.yml
├── networking.yml
├── properties.yml
├── README
└── scaling.yml

0 directories, 10 files
```
Lets make the manifest
```
~/ops/concourse-deployments$ cd aws/proto/
~/ops/concourse-deployments/aws/proto$ make manifest
11 error(s) detected:
 - $.compilation.cloud_properties.availability_zone: What availability zone should your concourse VMs be in?
 - $.jobs.haproxy.templates.haproxy.properties.ha_proxy.ssl_pem: Want ssl? define a pem
 - $.jobs.web.templates.atc.properties.external_url: What is the external URL for this concourse?
 - $.meta.availability_zone: What availability zone should your concourse VMs be in?
 - $.meta.external_url: What is the external URL for this concourse?
 - $.meta.ssl_pem: Want ssl? define a pem
 - $.networks.concourse.subnets: Specify your concourse subnet
 - $.resource_pools.db.cloud_properties.availability_zone: What availability zone should your concourse VMs be in?
 - $.resource_pools.haproxy.cloud_properties.availability_zone: What availability zone should your concourse VMs be in?
 - $.resource_pools.web.cloud_properties.availability_zone: What availability zone should your concourse VMs be in?
 - $.resource_pools.workers.cloud_properties.availability_zone: What availability zone should your concourse VMs be in?


Failed to merge templates; bailing...
Makefile:22: recipe for target 'manifest' failed
make: *** [manifest] Error 5
~/ops/concourse-deployments/aws/proto$
```

Again starting with Meta lines:

```
~/ops/concourse-deployments/aws/proto$ cat properties.yml
---
meta:
  availability_zone: "us-west-2a"   # Set this to match your first zone "aws_az1"
  external_url: "https://ci.x.x.x.x.sslip.io"  # Set as IP address of the Bastion host to allow testing via SSH tunnel
  ssl_pem: ~
  #  ssl_pem: (( vault meta.vault_prefix "/web_ui:pem" ))
```

Be sure to replace the x.x.x.x in the external_url above with the IP address of the Bastion host.

The `~` means we won't use SSL certs for now.  If you have proper certs or want to use self signed you can add them to vault under the `web_ui:pem` key

For networking, we put this inside Global Infrastructure or Site Infrastructure
```
~/ops/concourse-deployments/aws/proto$ cat networking.yml
---
networks:
  - name: concourse
    subnets:
      - range: 10.4.1.0/24
        gateway: 10.4.1.1
        dns:     [10.4.1.2]
        static:
          - 10.4.1.48 - 10.4.1.56  # We use 48-64, reserving the first eight for static
        reserved:
          - 10.4.1.2 - 10.4.1.3    # Amazon reserves these
		  - 10.4.1.4 - 10.4.1.47   # Allocated to other deployments
          - 10.4.1.65 - 10.4.1.254 # Allocated to other deployments
        cloud_properties:
          subnet: subnet-nnnnnnnn # <-- your AWS Subnet ID
          security_groups: [wide-open]
```

After it is deployed, you can do a quick test by hitting the HAProxy machine

```
~/ops/concourse-deployments/aws/proto$ bosh vms aws-proto-concourse
Acting as user 'admin' on deployment 'aws-proto-concourse' on 'aws-proto-bosh'

Director task 43

Task 43 done

+--------------------------------------------------+---------+-----+---------+------------+
| VM                                               | State   | AZ  | VM Type | IPs        |
+--------------------------------------------------+---------+-----+---------+------------+
| db/0 (fdb7a556-e285-4cf0-8f35-e103b96eff46)      | running | n/a | db      | 10.4.1.61  |
| haproxy/0 (5318df47-b138-44d7-b3a9-8a2a12833919) | running | n/a | haproxy | 10.4.1.51  |
| web/0 (ecb71ebc-421d-4caa-86af-81985958578b)     | running | n/a | web     | 10.4.1.48  |
| worker/0 (c2c081e0-c1ef-4c28-8c7d-ff589d05a1aa)  | running | n/a | workers | 10.4.1.62  |
| worker/1 (12a4ae1f-02fc-4c3b-846b-ae232215c77c)  | running | n/a | workers | 10.4.1.57  |
| worker/2 (b323f3ba-ebe4-4576-ab89-1bce3bc97e65)  | running | n/a | workers | 10.4.1.58  |
+--------------------------------------------------+---------+-----+---------+------------+

VMs total: 6
~/ops/concourse-deployments/aws/proto$ curl -i 10.4.1.51
HTTP/1.1 200 OK
Date: Thu, 07 Jul 2016 04:50:05 GMT
Content-Type: text/html; charset=utf-8
Transfer-Encoding: chunked

<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Concourse</title>
```

You can then run on a your local machine

```
$ ssh -L 8080:10.4.1.51:80 user@ci.x.x.x.x.sslip.io
```

and hit http://localhost:8080 to get the Concourse UI. Be sure to replace `user` with the jumpbox username on the Bastion host
and x.x.x.x with the IP address of the Bastion host.

# Building out Sites and Environments

Now that the underlying infrastructure has been deployed, we can start deplying our alpha/beta/other sites, with Cloud Foundry, and any required services. When using Concourse to update BOSH deployments,
there are the concepts of `alpha` and `beta` sites. The alpha site is the initial place where all deployment changes are checked for sanity + deployability. Typically this is done with a `bosh-lite` VM. The `beta` sites are where site-level changes are vetted. Usually these are referred to as the sandbox or staging environments, and there will be one per site, by necessity. Once changes have passed both the alpha, and beta site, we know it is safe for them to be rolled out to other sites, like production.

## Alpha

### BOSH-Lite

Since our `alpha` site will be a bosh lite running on AWS, we will need to deploy that to our [global infrastructure network][netplan].

First, lets make sure we're in the right place, targetting the right Vault:

```
$ cd ~/ops
$ safe target ops
Now targeting ops at https://10.4.1.16:8200
```

Now we can create our repo for deploying the bosh-lite:

```
$ genesis new deployment --template bosh-lite
cloning from template https://github.com/starkandwayne/bosh-lite-deployment
Cloning into '/home/gfranks/ops/bosh-lite-deployments'...
remote: Counting objects: 55, done.
remote: Compressing objects: 100% (33/33), done.
remote: Total 55 (delta 7), reused 55 (delta 7), pack-reused 0
Unpacking objects: 100% (55/55), done.
Checking connectivity... done.
Embedding genesis script into repository
genesis v1.5.2 (ec9c868f8e62)
[master 5421665] Initial clone of templated bosh-lite deployment
 3 files changed, 3672 insertions(+), 67 deletions(-)
  rewrite README.md (96%)
   create mode 100755 bin/genesis
```

Next lets create our site and environment:

```
$ cd bosh-lite-deployments
$ genesis new site --template aws aws
Created site aws (from template aws):
/home/gfranks/ops/bosh-lite-deployments/aws
├── README
└── site
    ├── disk-pools.yml
    ├── jobs.yml
    ├── networks.yml
    ├── properties.yml
    ├── README
    ├── releases
    ├── resource-pools.yml
    ├── stemcell
    │   ├── name
    │   └── version
    └── update.yml

2 directories, 11 files

$ genesis new env bosh-lite alpha
Running env setup hook: /home/gfranks/ops/bosh-lite-deployments/.env_hooks/setup

(*) ops	https://10.4.1.16:8200

Use this Vault for storing deployment credentials?  [yes or no]yes
Setting up credentials in vault, under secret/aws/alpha/bosh-lite
.
└── secret/aws/alpha/bosh-lite
    ├── blobstore/


    │   ├── agent
    │   └── director
    ├── db
    ├── nats
    ├── users/
    │   ├── admin
    │   └── hm
    └── vcap




Created environment aws/alpha:
/home/gfranks/ops/bosh-lite-deployments/aws/alpha
├── cloudfoundry.yml
├── credentials.yml
├── director.yml
├── Makefile


├── monitoring.yml
├── name.yml
├── networking.yml
├── properties.yml
├── README
└── scaling.yml

0 directories, 10 files

```

Now lets try to deploy:

```
$ make deploy
  checking https://genesis.starkandwayne.com for details on latest stemcell bosh-aws-xen-hvm-ubuntu-trusty-go_agent
  checking https://genesis.starkandwayne.com for details on release bosh/256.2
  checking https://genesis.starkandwayne.com for details on release bosh-warden-cpi/29
  checking https://genesis.starkandwayne.com for details on release garden-linux/0.339.0
  checking https://genesis.starkandwayne.com for details on release port-forwarding/2
8 error(s) detected:
 - $.meta.aws.azs.z1: What Availability Zone will BOSH be in?
 - $.meta.net.dns: What is the IP of the DNS server for this BOSH-Lite?
 - $.meta.net.gateway: What is the gateway of the network the BOSH-Lite will be on?
 - $.meta.net.range: What is the network address of the subnet BOSH-Lite will be on?
 - $.meta.net.reserved: Provide a list of reserved IP ranges for the subnet that BOSH-Lite will be on
 - $.meta.net.security_groups: What security groups should be applied to the BOSH-Lite?
 - $.meta.net.static: Provide a list of static IPs/ranges in the subnet that BOSH-Lite will choose from
 - $.meta.port_forwarding_rules: Define any port forwarding rules you wish to enable on the bosh-lite, or an empty array


Failed to merge templates; bailing...


Makefile:25: recipe for target 'deploy' failed
make: *** [deploy] Error 3
```

Looks like we only have a handful of parameters to update, all related to networking, so lets fill out our `networking.yml`,
after consulting the [Network Plan][netplan] to find our global infrastructure network and the AWS console to find our subnet
ID:

```
$ cat networking.yml
---
meta:
  net:
    subnet: subnet-xxxxx # <--- your subnet ID here
    security_groups: [wide-open]
    range: 10.4.1.0/24
    gateway: 10.4.1.1
    dns: [10.4.0.2]
  aws:
    azs:
      z1: us-west-2a
```

Since there are a bunch of other deployments on the infrastructure network, we should take care
to reserve the correct static + reserved IPs, so that we don't conflict with other deployments. Fortunately
that data can be referenced in the [Global Infrastructure IP Allocation section][infra-ips] of the Network Plan:

```
$ cat networking.yml
---
meta:
  net:
    subnet: subnet-8a186dee
    security_groups: [wide-open]
    range: 10.4.1.0/24
    gateway: 10.4.1.1
    static: [10.4.1.80]
    reserved: [10.4.1.2 - 10.4.1.79, 10.4.1.96 - 10.4.1.255]
    dns: [10.4.0.2]
  aws:
    azs:
      z1: us-west-2a
```

Lastly, we will need to add port-forwarding rules, so that things outside the bosh-lite can talk to its services.
Since we know we will be deploying Cloud Foundry, let's add rules for it:

```
$ cat properties.yml
---
meta:
  port_forwarding_rules:
  - internal_ip: 10.244.0.34
    internal_port: 80
    external_port: 80
  - internal_ip: 10.244.0.34
    internal_port: 443
    external_port: 443
```

And finally, we can deploy again:

```
$ make deploy
  checking https://genesis.starkandwayne.com for details on stemcell bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3262.2
    checking https://genesis.starkandwayne.com for details on release bosh/256.2
  checking https://genesis.starkandwayne.com for details on release bosh-warden-cpi/29
    checking https://genesis.starkandwayne.com for details on release garden-linux/0.339.0
  checking https://genesis.starkandwayne.com for details on release port-forwarding/2
    checking https://genesis.starkandwayne.com for details on stemcell bosh-aws-xen-hvm-ubuntu-trusty-go_agent/3262.2
  checking https://genesis.starkandwayne.com for details on release bosh/256.2
    checking https://genesis.starkandwayne.com for details on release bosh-warden-cpi/29
  checking https://genesis.starkandwayne.com for details on release garden-linux/0.339.0
    checking https://genesis.starkandwayne.com for details on release port-forwarding/2
Acting as user 'admin' on 'aws-proto-bosh'
Checking whether release bosh/256.2 already exists...YES
Acting as user 'admin' on 'aws-proto-bosh'
Checking whether release bosh-warden-cpi/29 already exists...YES
Acting as user 'admin' on 'aws-proto-bosh'
Checking whether release garden-linux/0.339.0 already exists...YES
Acting as user 'admin' on 'aws-proto-bosh'
Checking whether release port-forwarding/2 already exists...YES
Acting as user 'admin' on 'aws-proto-bosh'
Checking if stemcell already exists...
Yes
Acting as user 'admin' on deployment 'aws-alpha-bosh-lite' on 'aws-proto-bosh'
Getting deployment properties from director...
Unable to get properties list from director, trying without it...

Detecting deployment changes
...
Deploying
---------
Are you sure you want to deploy? (type 'yes' to continue): yes

Director task 58
  Started preparing deployment > Preparing deployment. Done (00:00:00)
...
Task 58 done

Started		2016-07-14 19:14:31 UTC
Finished	2016-07-14 19:17:42 UTC
Duration	00:03:11

Deployed `aws-alpha-bosh-lite' to `aws-proto-bosh'
```

Now we can verify the deployment and set up our `bosh` CLI target:

```
# grab the admin password for the bosh-lite
$ safe get secret/aws/alpha/bosh-lite/users/admin
--- # secret/aws/alpha/bosh-lite/users/admin
password: YOUR-PASSWORD-WILL-BE-HERE


$ bosh target https://10.4.1.80:25555 alpha
Target set to `aws-alpha-bosh-lite'
Your username: admin
Enter password:
Logged in as `admin'
$ bosh status
Config
             /home/gfranks/.bosh_config

 Director
   Name       aws-alpha-bosh-lite
     URL        https://10.4.1.80:25555
   Version    1.3232.2.0 (00000000)
     User       admin
   UUID       d0a12392-f1df-4394-99d1-2c6ce376f821
     CPI        vsphere_cpi
   dns        disabled
     compiled_package_cache disabled
   snapshots  disabled

   Deployment
     not set
```

Tadaaa! Time to commit all the changes to deployment repo, and push to where we're storing
them long-term.

### Cloud Foundry

To deploy CF to our alpha environment, we will need to first ensure we're targeting the right
Vault/BOSH:

```
$ cd ~/ops
$ safe target ops

(*) ops	https://10.4.1.16:8200

$ bosh target alpha
Target set to `aws-alpha-bosh-lite'
```

Now we'll create our deployment repo for cloudfoundry:

```
$ genesis new deployment --template cf`
cloning from template https://github.com/starkandwayne/cf-deployment
Cloning into '/home/gfranks/ops/cf-deployments'...
remote: Counting objects: 268, done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 268 (delta 0), reused 0 (delta 0), pack-reused 265
Receiving objects: 100% (268/268), 51.57 KiB | 0 bytes/s, done.
Resolving deltas: 100% (112/112), done.
Checking connectivity... done.
Embedding genesis script into repository
genesis v1.5.2 (ec9c868f8e62)
[master 1f0c534] Initial clone of templated cf deployment
 2 files changed, 3666 insertions(+), 150 deletions(-)
 rewrite README.md (99%)
 create mode 100755 bin/genesis
```

And generate our bosh-lite based alpha environment:

```
$ cf cf-deployments
$ genesis new site --template bosh-lite bosh-lite
Created site bosh-lite (from template bosh-lite):
/home/gfranks/ops/cf-deployments/bosh-lite
├── README
└── site
    ├── disk-pools.yml
    ├── jobs.yml
    ├── networks.yml
    ├── properties.yml
    ├── releases
    ├── resource-pools.yml
    ├── stemcell
    │   ├── name
    │   └── version
    └── update.yml

2 directories, 10 files

$ genesis new env bosh-lite alpha
Running env setup hook: /home/gfranks/ops/cf-deployments/.env_hooks/00_confirm_vault

(*) ops	https://10.4.1.16:8200

Use this Vault for storing deployment credentials?  [yes or no] yes
Running env setup hook: /home/gfranks/ops/cf-deployments/.env_hooks/setup_certs
Generating Cloud Foundry internal certs
Uploading Cloud Foundry internal certs to Vault
Running env setup hook: /home/gfranks/ops/cf-deployments/.env_hooks/setup_cf_secrets
Creating JWT Signing Key
Creating app_ssh host key fingerprint
Generating secrets
Created environment bosh-lite/alpha:
/home/gfranks/ops/cf-deployments/bosh-lite/alpha
├── cloudfoundry.yml
├── credentials.yml
├── director.yml
├── Makefile
├── monitoring.yml
├── name.yml
├── networking.yml
├── properties.yml
├── README
└── scaling.yml

0 directories, 10 files


```

Unlike all the other deployments so far, we won't use `make deploy` to vet the
manifest for CF. This is because the bosh-lite CF comes out of the box ready to
deploy to a Vagrant-based bosh-lite with no tweaks.  Since we are using it as
the Cloud Foundry for our alpha environment, we will need to customize the Cloud
Foundry base domain, with a domain resolving to the IP of our `alpha` bosh-lite VM:

```
cd bosh-lite/alpha
$ cat properties.yml
---
meta:
  cf:
    base_domain: 10.4.1.80.xip.io
```

Now we can deploy:

```
$ make deploy
  checking https://genesis.starkandwayne.com for details on release cf/237
  checking https://genesis.starkandwayne.com for details on release toolbelt/3.2.10
  checking https://genesis.starkandwayne.com for details on release postgres/1.0.3
  checking https://genesis.starkandwayne.com for details on release cf/237
  checking https://genesis.starkandwayne.com for details on release toolbelt/3.2.10
  checking https://genesis.starkandwayne.com for details on release postgres/1.0.3
Acting as user 'admin' on 'aws-try-anything-bosh-lite'
Checking whether release cf/237 already exists...NO
Using remote release `https://bosh.io/d/github.com/cloudfoundry/cf-release?v=237'

Director task 1
  Started downloading remote release > Downloading remote release
...
Deploying
---------
Are you sure you want to deploy? (type 'yes' to continue): yes

Director task 12
  Started preparing deployment > Preparing deployment. Done (00:00:01)
...
Task 12 done

Started		2016-07-15 14:47:45 UTC
Finished	2016-07-15 14:51:28 UTC
Duration	00:03:43

Deployed `bosh-lite-alpha-cf' to `aws-try-anything-bosh-lite'
```

And once complete, run the smoke tests for good measure:

```
$ genesis bosh run errand smoke_tests
FIXME output
```

We now have our alpha-environment's Cloud Foundry stood up!

## First Beta Environment

Now that our `alpha` environment has been deployed, we can deploy our first beta environment to AWS. To do this, we will first deploy a BOSH director for the environment using the `bosh-deployments` repo we generated back when we built our [Proto BOSH](#proto-bosh), and then deploy Cloud Foundry on top of it.

### BOSH
```
$ cd ~/ops/bosh-deployments
$ bosh target proto
$ ls
aws  bin  global  LICENSE  README.md
```

We already have the `aws` site created, so now we will just need to create our new environment, and deploy it:

```
$ safe target ops
Now targeting ops at http://10.10.10.6:8200
$ genesis new env aws staging
RSA 1024 bit CA certificates are loaded due to old openssl compatibility
Running env setup hook: /home/centos/ops/bosh-deployments/.env_hooks/setup

 ops	http://10.10.10.6:8200

Use this Vault for storing deployment credentials?  [yes or no] yes
Setting up credentials in vault, under secret/aws/staging/bosh
.
└── secret/aws/staging/bosh
    ├── blobstore/
    │   ├── agent
    │   └── director
    ├── db
    ├── nats
    ├── users/
    │   ├── admin
    │   └── hm
    └── vcap


Created environment aws/staging:
/home/centos/ops/bosh-deployments/aws/staging
├── cloudfoundry.yml
├── credentials.yml
├── director.yml
├── Makefile
├── monitoring.yml
├── name.yml
├── networking.yml
├── properties.yml
├── README
└── scaling.yml

0 directories, 10 files

```

Notice, unlike the Proto BOSH setup, we do not specify `--type bosh-init`. This means we will use BOSH itself (in this case the Proto-BOSH) to deploy our sandbox BOSH. Again, the environment hook created all of our credentials for us, but this time we targeted the long-term Vault, so there will be no need for migrating credentials around.

Let's try to deploy now, and see what information still needs to be resolved:

```
$ cd aws/staging
$ make deploy
9 error(s) detected:
 - $.meta.aws.access_key: Please supply an AWS Access Key
 - $.meta.aws.azs.z1: What Availability Zone will BOSH be in?
 - $.meta.aws.default_sgs: What security groups should VMs be placed in, if none are specified in the deployment manifest?
 - $.meta.aws.private_key: What private key will be used for establishing the ssh_tunnel (bosh-init only)?
 - $.meta.aws.region: What AWS region are you going to use?
 - $.meta.aws.secret_key: Please supply an AWS Secret Key
 - $.meta.aws.ssh_key_name: What AWS keypair should be used for the vcap user?
 - $.meta.shield_public_key: Specify the SSH public key from this environment's SHIELD daemon
 - $.networks.default.subnets: Specify subnets for your BOSH vm's network


Failed to merge templates; bailing...
make: *** [deploy] Error 3
```

Looks like we need to provide the same type of data as we did for Proto BOSH. Lets fill in the basic properties:

```
$ cat > properties.yml <<EOF
---
meta:
  aws:
    access_key: (( vault "secret/aws:access_key" ))
    secret_key: (( vault "secret/aws:secret_key" ))
    private_key: ~ # not needed, since not using bosh-lite
    region: us-west-2
    azs:
      z1: (( concat meta.aws.region "a")
    ssh_key_name: your-ec2-keypair-name
    default_sgs: [wide-open]
  shield_public_key: (( vault "secret/aws/proto/shield/keys/core:public" ))
EOF
```

This was a bit easier than it was for Proto BOSH, since our SHIELD public key exists now, and our
AWS keys are already in Vault.

Verifying our changes worked, we see that we only need to provide networking configuration at this point:

```
make deploy
$ make deploy
2 error(s) detected:
 - $.meta.aws.default_sgs: What security groups should VMs be placed in, if none are specified in the deployment manifest?
 - $.networks.default.subnets: Specify subnets for your BOSH vm's network


Failed to merge templates; bailing...
make: *** [deploy] Error 3

```

All that remains is filling in our networking details, so lets go consult our [Network Plan](https://github.com/starkandwayne/codex/blob/master/network.md). We will place the BOSH director in the staging site's infrastructure network, in the first AZ we have defined (subnet name `staging-infra-1`, CIDR `10.4.32.0/24`). To do that, we'll need to update `networking.yml`:

```
$ cat > networking.yml <<EOF
---
networks:
  - name: default
    subnets:
      - range:    10.4.32.0/24
        gateway:  10.4.32.1
        dns:     [10.4.0.2]
        cloud_properties:
          subnet: subnet-xxxxxxxx # <-- the AWS Subnet ID for your staging-infra-1 network
          security_groups: [wide-open]
        reserved:
          - 10.4.32.2 - 10.4.32.3    # Amazon reserves these
            # BOSH is in 10.4.32.0/28
          - 10.4.32.16 - 10.4.32.254 # Allocated to other deployments
        static:
          - 10.4.32.4
EOF
```

Now that that's handled, let's deploy for real:

```
$ make deploy
$ make deploy
RSA 1024 bit CA certificates are loaded due to old openssl compatibility
Acting as user 'admin' on 'aws-proto-bosh-microboshen-aws'
Checking whether release bosh/256.2 already exists...YES
Acting as user 'admin' on 'aws-proto-bosh-microboshen-aws'
Checking whether release bosh-aws-cpi/53 already exists...YES
Acting as user 'admin' on 'aws-proto-bosh-microboshen-aws'
Checking whether release shield/6.2.1 already exists...YES
Acting as user 'admin' on 'aws-proto-bosh-microboshen-aws'
Checking if stemcell already exists...
Yes
Acting as user 'admin' on deployment 'aws-staging-bosh' on 'aws-proto-bosh-microboshen-aws'
Getting deployment properties from director...

Detecting deployment changes
----------------------------
resource_pools:
- cloud_properties:
    availability_zone: us-east-1b
    ephemeral_disk:
      size: 25000
      type: gp2
    instance_type: m3.xlarge
  env:
    bosh:
      password: "<redacted>"
  name: bosh
  network: default
  stemcell:
    name: bosh-aws-xen-hvm-ubuntu-trusty-go_agent
    sha1: 971e869bd825eb0a7bee36a02fe2f61e930aaf29
    url: https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent?v=3232.6
...
Deploying
---------
Are you sure you want to deploy? (type 'yes' to continue): yes

Director task 144
  Started preparing deployment > Preparing deployment. Done (00:00:00)

  Started preparing package compilation > Finding packages to compile. Done (00:00:00)
...
Task 144 done

Started		2016-07-08 17:23:47 UTC
Finished	2016-07-08 17:34:46 UTC
Duration	00:10:59

Deployed 'aws-staging-bosh' to 'aws-proto-bosh'
```

This will take a little less time than Proto BOSH did (some packages were already compiled), and the next time you deploy, it go by much quicker, as all the packages should have been compiled by now (unless upgrading BOSH or the stemcell).

Once the deployment finishes, target the new BOSH director to verify it works:

```
$ safe get secret/aws/sandbox/bosh/users/admin # grab the admin user's password for bosh
$ bosh target https://10.4.32.4:25555 aws-sandbox
Target set to 'aws-staging-bosh'
Your username: admin
Enter password:
Logged in as 'admin'
```

Again, since our creds are already in the long-term vault, we can skip the credential migratoin that was done in the proto-bosh deployment and go straight to committing our new deployment to the repo, and pushing it upstream.

Now it's time to move on to deploying our `beta` (sandbox) Cloud Foundry!

### Jumpboxen?

### Cloud Foundry

To deploy Cloud Foundry, we will go back into our ops directory, making use of `cf-deplyoments` repo
created when we built our alpha site:

```
$ cd ~/ops/cf-deployments
```

Also, make sure that you're targeting the right Vault, for good measure:

```
$ safe target ops
```

We will now create an `aws` site for CF:

```
$ genesis new site --template aws aws
Created site aws (from template aws):
/home/centos/ops/cf-deployments/aws
├── README
└── site
    ├── disk-pools.yml
    ├── jobs.yml
    ├── networks.yml
    ├── properties.yml
    ├── releases
    ├── resource-pools.yml
    ├── stemcell
    │   ├── name
    │   └── version
    └── update.yml

2 directories, 10 files

```

And the `staging` environment inside it:

```
$ genesis new env aws staging
RSA 1024 bit CA certificates are loaded due to old openssl compatibility
Running env setup hook: /home/centos/ops/cf-deployments/.env_hooks/00_confirm_vault

 ops	http://10.10.10.6:8200

Use this Vault for storing deployment credentials?  [yes or no] yes
Running env setup hook: /home/centos/ops/cf-deployments/.env_hooks/setup_certs
Generating Cloud Foundry internal certs
Uploading Cloud Foundry internal certs to Vault
Running env setup hook: /home/centos/ops/cf-deployments/.env_hooks/setup_cf_secrets
Creating JWT Signing Key
Creating app_ssh host key fingerprint
Generating secrets
Created environment aws/staging:
/home/centos/ops/cf-deployments/aws/staging
├── cloudfoundry.yml
├── credentials.yml
├── director.yml
├── Makefile
├── monitoring.yml
├── name.yml
├── networking.yml
├── properties.yml
├── README
└── scaling.yml

0 directories, 10 files

```

As you might have guessed, the next step will be to see what parameters we need to fill in:

```
$ make deploy
$ make deploy
57 error(s) detected:
 - $.meta.azs.z1: What availability zone should the *_z1 vms be placed in?
 - $.meta.azs.z2: What availability zone should the *_z2 vms be placed in?
 - $.meta.azs.z3: What availability zone should the *_z3 vms be placed in?
 - $.meta.cf.base_domain: Enter the Cloud Foundry base domain
 - $.meta.cf.blobstore_config.fog_connection.aws_access_key_id: What is the access key id for the blobstore S3 buckets?
 - $.meta.cf.blobstore_config.fog_connection.aws_secret_access_key: What is the secret key for the blobstore S3 buckets?
 - $.meta.cf.blobstore_config.fog_connection.region: Which region are the blobstore S3 buckets in?
 - $.meta.cf.ccdb.host: What hostname/IP is the ccdb available at?
 - $.meta.cf.ccdb.pass: Specify the password of the ccdb user
 - $.meta.cf.ccdb.user: Specify the user to connect to the ccdb
 - $.meta.cf.uaadb.host: What hostname/IP is the uaadb available at?
 - $.meta.cf.uaadb.pass: Specify the password of the uaadb user
 - $.meta.cf.uaadb.user: Specify the user to connect to the uaadb
 - $.meta.dns: Enter the DNS server for your VPC
 - $.meta.elbs: What elbs will be in front of the gorouters?
 - $.meta.router_security_groups: Enter the security groups which should be applied to the gorouter VMs
 - $.meta.security_groups: Enter the security groups which should be applied to CF VMs
 - $.networks.cf1.subnets.0.cloud_properties.subnet: Enter the AWS subnet ID for this subnet
 - $.networks.cf1.subnets.0.gateway: Enter the Gateway for this subnet
 - $.networks.cf1.subnets.0.range: Enter the CIDR address for this subnet
 - $.networks.cf1.subnets.0.reserved: Enter the reserved IP ranges for this subnet
 - $.networks.cf1.subnets.0.static: Enter the static IP ranges for this subnet
 - $.networks.cf2.subnets.0.cloud_properties.subnet: Enter the AWS subnet ID for this subnet
 - $.networks.cf2.subnets.0.gateway: Enter the Gateway for this subnet
 - $.networks.cf2.subnets.0.range: Enter the CIDR address for this subnet
 - $.networks.cf2.subnets.0.reserved: Enter the reserved IP ranges for this subnet
 - $.networks.cf2.subnets.0.static: Enter the static IP ranges for this subnet
 - $.networks.cf3.subnets.0.cloud_properties.subnet: Enter the AWS subnet ID for this subnet
 - $.networks.cf3.subnets.0.gateway: Enter the Gateway for this subnet
 - $.networks.cf3.subnets.0.range: Enter the CIDR address for this subnet
 - $.networks.cf3.subnets.0.reserved: Enter the reserved IP ranges for this subnet
 - $.networks.cf3.subnets.0.static: Enter the static IP ranges for this subnet
 - $.networks.router1.subnets.0.cloud_properties.subnet: Enter the AWS subnet ID for this subnet
 - $.networks.router1.subnets.0.gateway: Enter the Gateway for this subnet
 - $.networks.router1.subnets.0.range: Enter the CIDR address for this subnet
 - $.networks.router1.subnets.0.reserved: Enter the reserved IP ranges for this subnet
 - $.networks.router1.subnets.0.static: Enter the static IP ranges for this subnet
 - $.networks.router2.subnets.0.cloud_properties.subnet: Enter the AWS subnet ID for this subnet
 - $.networks.router2.subnets.0.gateway: Enter the Gateway for this subnet
 - $.networks.router2.subnets.0.range: Enter the CIDR address for this subnet
 - $.networks.router2.subnets.0.reserved: Enter the reserved IP ranges for this subnet
 - $.networks.router2.subnets.0.static: Enter the static IP ranges for this subnet
 - $.properties.cc.buildpacks.fog_connection.aws_access_key_id: What is the access key id for the blobstore S3 buckets?
 - $.properties.cc.buildpacks.fog_connection.aws_secret_access_key: What is the secret key for the blobstore S3 buckets?
 - $.properties.cc.buildpacks.fog_connection.region: Which region are the blobstore S3 buckets in?
 - $.properties.cc.droplets.fog_connection.aws_access_key_id: What is the access key id for the blobstore S3 buckets?
 - $.properties.cc.droplets.fog_connection.aws_secret_access_key: What is the secret key for the blobstore S3 buckets?
 - $.properties.cc.droplets.fog_connection.region: Which region are the blobstore S3 buckets in?
 - $.properties.cc.packages.fog_connection.aws_access_key_id: What is the access key id for the blobstore S3 buckets?
 - $.properties.cc.packages.fog_connection.aws_secret_access_key: What is the secret key for the blobstore S3 buckets?
 - $.properties.cc.packages.fog_connection.region: Which region are the blobstore S3 buckets in?
 - $.properties.cc.resource_pool.fog_connection.aws_access_key_id: What is the access key id for the blobstore S3 buckets?
 - $.properties.cc.resource_pool.fog_connection.aws_secret_access_key: What is the secret key for the blobstore S3 buckets?
 - $.properties.cc.resource_pool.fog_connection.region: Which region are the blobstore S3 buckets in?
 - $.properties.cc.security_group_definitions.load_balancer.rules: Specify the rules for allowing access for CF apps to talk to the CF Load Balancer External IPs
 - $.properties.cc.security_group_definitions.services.rules: Specify the rules for allowing access to CF services subnets
 - $.properties.cc.security_group_definitions.user_bosh_deployments.rules: Specify the rules for additional BOSH user services that apps will need to talk to


Failed to merge templates; bailing...
make: *** [deploy] Error 3
```

Oh boy. That's a lot. Cloud Foundry must be compilicated. Looks like a lot of the fog_connection properties are all duplicates though, so lets fill out `properties.yml` with those:

```
$ cat properties.yml
---
meta:
  aws:
    fog_connection:
      aws_access_key_id: (( vault "secret/aws:access_key" ))
      aws_secret_access_key: (( vault "secret/aws:secret_key"))
      region: us-east-1
```

Next, lets tackle the database situation. We will need to create RDS instances for the `uaadb` and `ccdb`.

**TODO:** make a terraform repo/script for creating RDS instances for your and store their creds + address + dbname  in vault

Now that we have RDS instances, lets refer to them in our `properties.yml` file:

```
cat properties.yml
---
meta:
  cf:
    fog_connection:
      aws_access_key_id:     (( vault "secret/aws:access_key" ))
      aws_secret_access_key: (( vault "secret/aws:secret_key"))
      region: us-east-1
    ccdb:
      host: (( vault meta.vault_prefix "/ccdb:host" ))
      user: (( vault meta.vault_prefix "/ccdb:user" ))
      pass: (( vault meta.vault_prefix "/ccdb:password" ))
    uaadb:
      host: (( vault meta.vault_prefix "/uaadb:host" ))
      user: (( vault meta.vault_prefix "/uaadb:user" ))
      pass: (( vault meta.vault_prefix "/uaadb:password" ))
```

Lastly, let's make sure to add our Cloud Foundry domain to properties.yml:

```
---
meta:
  cf:
    domain: your.staging.cf.example.com
    fog_connection:
      aws_access_key_id:     (( vault "secret/aws:access_key" ))
      aws_secret_access_key: (( vault "secret/aws:secret_key"))
      region: us-east-1
    ccdb:
      host: (( vault meta.vault_prefix "/ccdb:host" ))
      user: (( vault meta.vault_prefix "/ccdb:user" ))
      pass: (( vault meta.vault_prefix "/ccdb:password" ))
    uaadb:
      host: (( vault meta.vault_prefix "/uaadb:host" ))
      user: (( vault meta.vault_prefix "/uaadb:user" ))
      pass: (( vault meta.vault_prefix "/uaadb:password" ))
```

And let's see what's left to fill out now:

```
$ make deploy
 - $.meta.azs.z1: What availability zone should the *_z1 vms be placed in?
 - $.meta.azs.z2: What availability zone should the *_z2 vms be placed in?
 - $.meta.azs.z3: What availability zone should the *_z3 vms be placed in?
 - $.meta.dns: Enter the DNS server for your VPC
 - $.meta.elbs: What elbs will be in front of the gorouters?
 - $.meta.router_security_groups: Enter the security groups which should be applied to the gorouter VMs
 - $.meta.security_groups: Enter the security groups which should be applied to CF VMs
 - $.networks.cf1.subnets.0.cloud_properties.subnet: Enter the AWS subnet ID for this subnet
 - $.networks.cf1.subnets.0.gateway: Enter the Gateway for this subnet
 - $.networks.cf1.subnets.0.range: Enter the CIDR address for this subnet
 - $.networks.cf1.subnets.0.reserved: Enter the reserved IP ranges for this subnet
 - $.networks.cf1.subnets.0.static: Enter the static IP ranges for this subnet
 - $.networks.cf2.subnets.0.cloud_properties.subnet: Enter the AWS subnet ID for this subnet
 - $.networks.cf2.subnets.0.gateway: Enter the Gateway for this subnet
 - $.networks.cf2.subnets.0.range: Enter the CIDR address for this subnet
 - $.networks.cf2.subnets.0.reserved: Enter the reserved IP ranges for this subnet
 - $.networks.cf2.subnets.0.static: Enter the static IP ranges for this subnet
 - $.networks.cf3.subnets.0.cloud_properties.subnet: Enter the AWS subnet ID for this subnet
 - $.networks.cf3.subnets.0.gateway: Enter the Gateway for this subnet
 - $.networks.cf3.subnets.0.range: Enter the CIDR address for this subnet
 - $.networks.cf3.subnets.0.reserved: Enter the reserved IP ranges for this subnet
 - $.networks.cf3.subnets.0.static: Enter the static IP ranges for this subnet
 - $.networks.router1.subnets.0.cloud_properties.subnet: Enter the AWS subnet ID for this subnet
 - $.networks.router1.subnets.0.gateway: Enter the Gateway for this subnet
 - $.networks.router1.subnets.0.range: Enter the CIDR address for this subnet
 - $.networks.router1.subnets.0.reserved: Enter the reserved IP ranges for this subnet
 - $.networks.router1.subnets.0.static: Enter the static IP ranges for this subnet
 - $.networks.router2.subnets.0.cloud_properties.subnet: Enter the AWS subnet ID for this subnet
 - $.networks.router2.subnets.0.gateway: Enter the Gateway for this subnet
 - $.networks.router2.subnets.0.range: Enter the CIDR address for this subnet
 - $.networks.router2.subnets.0.reserved: Enter the reserved IP ranges for this subnet
 - $.networks.router2.subnets.0.static: Enter the static IP ranges for this subnet
 - $.properties.cc.security_group_definitions.load_balancer.rules: Specify the rules for allowing access for CF apps to talk to the CF Load Balancer External IPs
 - $.properties.cc.security_group_definitions.services.rules: Specify the rules for allowing access to CF services subnets
 - $.properties.cc.security_group_definitions.user_bosh_deployments.rules: Specify the rules for additional BOSH user services that apps will need to talk to
```

All of those parameters look like they're networking related. Time to start building out the `networking.yml` file. Since our VPC is `10.4.0.0/16`, Amazon will have provided a DNS server for us at `10.4.0.2`. We can grab the AZs and ELB names from our terraform output, and define our router + cf security groups, without consulting the Network Plan:

```
$ cat networking.yml
---
meta:
  azs:
    z1: us-west-2a
    z2: us-west-2b
    z3: us-west-2c
  dns: 10.4.0.2
  elbs: [staging-cf-elb]
  router_security_groups: [wide-open]
  security_groups: [wide-open]
```

Now, we can consult our [Network Plan][netplan] for the subnet information,  cross referencing with terraform output or the AWS console to get the subnet ID:

```
$ cat networking.yml
---
---
meta:
  azs:
    z1: us-west-2a
    z2: us-west-2b
    z3: us-west-2c
  dns: 10.4.0.2
  elbs: [staging-cf-elb]
  router_security_groups: [wide-open]
  security_groups: [wide-open]

networks:
- name: cf1
  subnets:
  - range: 10.4.36.0/24
    static: [10.4.36.4 - 10.4.36.100]
    reserved: [10.4.36.2 - 10.4.36.3\ # amazon reserves these
    gateway: 10.4.36.1
    cloud_properties:
      subnet: subnet-XXXXXX # <--- your subnet ID here
- name: cf2
  - range: 10.4.37.0/24
    static: [10.4.37.4 - 10.4.37.100]
    reserved: [10.4.37.2 - 10.4.37.3] # amazon reserves these
    gateway: 10.4.37.1
    cloud_properties:
      subnet: subnet-XXXXXX # <--- your subnet ID here
- name: cf3
  - range: 10.4.38.0/24
    static: [10.4.38.4 - 10.4.38.100]
    reserved: [10.4.38.2 - 10.4.38.3] # amazon reserves these
    gateway: 10.4.38.1
    cloud_properties:
      subnet: subnet-XXXXXX # <--- your subnet ID here
- name: router1
  - range: 10.4.34.0/24
    static: [10.4.34.4 - 10.4.34.100]
    reserved: [10.4.34.2 - 10.4.34.3] # amazon reserves these
    gateway: 10.4.34.1
    cloud_properties:
      subnet: subnet-XXXXXX # <--- your subnet ID here
- name: router2
  - range: 10.4.35.0/24
    static: [10.4.35.4 - 10.4.35.100]
    reserved: [10.4.35.2 - 10.4.47.3] # amazon reserves these
    gateway: 10.4.35.1
    cloud_properties:
      subnet: subnet-XXXXXX # <--- your subnet ID here
```

Let's see what's left now:

```
$ make deploy
3 error(s) detected:
 - $.properties.cc.security_group_definitions.load_balancer.rules: Specify the rules for allowing access for CF apps to talk to the CF Load Balancer External IPs
 - $.properties.cc.security_group_definitions.services.rules: Specify the rules for allowing access to CF services subnets
 - $.properties.cc.security_group_definitions.user_bosh_deployments.rules: Specify the rules for additional BOSH user services that apps will need to talk to
```

The only bits left are the Cloud Foundry security group definitions (applied to each running app, not the SGs applied to the CF VMs). We add three sets of rules for apps to have access to by default - `load_balancer`, `services`, and `user_bosh_deployments`. The `load_balancer` group should have a rule allowing access to the public IP(s) of the Cloud Foundry installation, so that apps are able to talk to other apps. The `services` group should have rules allowing access to the internal IPs of the services networks (according to our [Network Plan][netplan], `10.4.42.0/24`, `10.4.43.0/24`, `10.4.44.0/24`). The `user_bosh_deployments` is used for any non-CF-services that the apps may need to talk to. In our case, there aren't any, so this can be an empty list.

```
$ cat networking.yml
---
meta:
  azs:
    z1: us-west-2a
    z2: us-west-2b
    z3: us-west-2c
  dns: 10.4.0.2
  elbs: [staging-cf-elb]
  router_security_group: [wide-open]
  security_groups: [wide-open]

networks:
- name: cf1
  subnets:
  - range: 10.4.36.0/24
    static: [10.4.36.4 - 10.4.36.100]
    reserved: [10.4.36.2 - 10.4.36.3\ # amazon reserves these
    gateway: 10.4.36.1
    cloud_properties:
      subnet: subnet-XXXXXX # <--- your subnet ID here
- name: cf2
  - range: 10.4.37.0/24
    static: [10.4.37.4 - 10.4.37.100]
    reserved: [10.4.37.2 - 10.4.37.3] # amazon reserves these
    gateway: 10.4.37.1
    cloud_properties:
      subnet: subnet-XXXXXX # <--- your subnet ID here
- name: cf3
  - range: 10.4.38.0/24
    static: [10.4.38.4 - 10.4.38.100]
    reserved: [10.4.38.2 - 10.4.38.3] # amazon reserves these
    gateway: 10.4.38.1
    cloud_properties:
      subnet: subnet-XXXXXX # <--- your subnet ID here
- name: router1
  - range: 10.4.34.0/24
    static: [10.4.34.4 - 10.4.34.100]
    reserved: [10.4.34.2 - 10.4.34.3] # amazon reserves these
    gateway: 10.4.34.1
    cloud_properties:
      subnet: subnet-XXXXXX # <--- your subnet ID here
- name: router2
  - range: 10.4.35.0/24
    static: [10.4.35.4 - 10.4.35.100]
    reserved: [10.4.35.2 - 10.4.47.3] # amazon reserves these
    gateway: 10.4.35.1
    cloud_properties:
      subnet: subnet-XXXXXX # <--- your subnet ID here

properties:
  cc:
    security_group_definitions:
    - name: load_balancer:
      rules:
      - destination: YOUR LOAD BALANCER IP1
        protocol: all
      - destination: YOUR LOAD BALANCER IP2
        protocol: all
    - name: services
      rules:
      - destination: 10.4.42.0-10.4.44.255
        protocol: all
    - name: user_bosh_deployments
      rules: []
```

That should be it, finally. Let's deploy!

```
$ make deploy
RSA 1024 bit CA certificates are loaded due to old openssl compatibility
Acting as user 'admin' on 'aws-staging-bosh'
Checking whether release cf/237 already exists...NO
Using remote release 'https://bosh.io/d/github.com/cloudfoundry/cf-release?v=237'

Director task 6
  Started downloading remote release > Downloading remote release
...
Deploying
---------
Are you sure you want to deploy? (type 'yes' to continue): yes
...

Started		2016-07-08 17:23:47 UTC
Finished	2016-07-08 17:34:46 UTC
Duration	00:10:59

Deployed 'aws-staging-cf' to 'aws-staging-bosh'

```

After a long while of compiling and deploying VMs, your CF should now be up, and accessible! You can
check the sanity of the deployment via `genesis bosh run errand smoke_tests`. Target it using
`cf login -a https://api.system.<your CF domain>`. The admin user's password can be retrieved
from Vault. If you run into any trouble, make sure that your DNS is pointing properly to the
correct ELB for this environment, and that the ELB has the correct SSL certificate for your site.

## Production Environment

Deploying the production environment will be much like deploying the `beta` environment above. You will need to deploy a BOSH director, Cloud Foundry, and any services also deployed in the `beta` site. Hostnames, credentials, network information, and possibly scaling parameters will all be different, but the procedure for deploying them is the same.


## Next Steps

Lather, rinse, repeat for all additional environments (dev, prod, loadtest, whatever's applicable to the client).



[aws-subnets]: http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html
[bolo]:        https://github.com/cloudfoundry-community/bolo-boshrelease
[cfconsul]:    https://docs.cloudfoundry.org/concepts/architecture/#bbs-consul
[cfetcd]:      https://docs.cloudfoundry.org/concepts/architecture/#etcd
[DRY]:         https://en.wikipedia.org/wiki/Don%27t_repeat_yourself
[jumpbox]:     https://github.com/starkandwayne/jumpbox
[netplan]:     https://github.com/starkandwayne/codex/blob/master/part3/network.md
[infra-ips]:   https://github.com/starkandwayne/codex/blob/master/part3/network.md#global-infrastructure-ip-allocation
[spruce-129]:  https://github.com/geofffranks/spruce/issues/129
[slither]:     http://slither.io
[amazon-keys]: https://console.aws.amazon.com/ec2/v2/home?#KeyPairs:sort=keyName
[az]:          http://aws.amazon.com/about-aws/global-infrastructure/