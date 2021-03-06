# BOSH tutorial


## Prerequisites

* laptop with administrator access

* comfortable using command line (as no GUI version of the tutorial is available, and Windows support is limited).

* your own AWS account
  * [Signing up for the AWS free tier](https://aws.amazon.com/free/)
	* Requires amazon login, credit card, mailing address, and phone number
	* Can take up to 24 hours for full activation
  * [Getting your AWS access and secret keys](http://docs.aws.amazon.com/general/latest/gr/managing-aws-access-keys.html)
	* Setting up an IAM user & giving the user administrative permissions
	* Downloading credentials
	* AWS cli
		* install homebrew: http://brew.sh/
		* `brew install awscli`
		* `aws configure`
		* `aws iam list-users`
	* Sign up for VPC

* an SSH keypair to use with AWS
	* You can use an existing key if you prefer, but the terraform scripts assume specific naming
```
$ ssh-keygen -t rsa -b 4096 -C "yourname@example.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/yourname/.ssh/id_rsa): /Users/yourname/.ssh/boshtutorial
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /Users/yourname/.ssh/boshtutorial.
Your public key has been saved in /Users/yourname/.ssh/boshtutorial.pub.
```

* [Terraform installed](https://www.terraform.io/downloads.html)
  * makes it easier to stand up AWS infrastructure
	* Download, extract zip file
        * Put contents in your PATH:

	`mv ~/Downloads/terraform_0.6.16_darwin_amd64/* /usr/local/bin`

	(or: brew install terraform)

* Clone this repo:
` git clone https://github.com/bridgetkromhout/bosh-tutorial`

----

## Overview

 What is BOSH? Brief overview
  * "BOSH is an open source tool for release engineering, deployment, lifecycle management, and monitoring of distributed systems."
	* Cloud Provider Interface (CPI) - IaaS-agnostic
	* Create VMs, attach storage, etc.
		* [AWS CPI github repo](https://github.com/cloudfoundry-incubator/bosh-aws-cpi-release)
		* [AWS CPI docs](https://bosh.io/docs/aws-cpi.html)
	* 1 BOSH director, N VMs running BOSH agents
	* For this tutorial, we're using [bosh-lite](https://github.com/cloudfoundry/bosh-lite) to simulate having multiple VMs.


  * http://bosh.io/docs/about.html
  * BOSH components: http://bosh.io/docs/bosh-components.html

## AWS infrastructure

* Standing up AWS infrastructure to use with BOSH
  * Terraform
	* run commands from the bosh-tutorial/terraform subdirectory

	`terraform plan` - see what we're going to launch

	`terraform apply` - launch AWS infrastructure

	`terraform destroy` - remove all terraform-managed infrastructure

* Launching bosh-lite instance

  * Looking at how the instance was created (Packer)
    * https://www.packer.io/docs/builders/amazon-ebs.html
    * https://github.com/cloudfoundry/bosh-lite/blob/master/packer/build-aws
    * https://www.terraform.io/docs/providers/aws/r/instance.html#root_block_device

* More about your AWS instance: `curl http://169.254.169.254/latest/meta-data/`

## Explore

The BOSH CLI
Communicates with the BOSH director
Use `bosh help` to view available commands
Verify that the BOSH CLI has a target already set: `bosh target`
the CLI and BOSH director are on a single VM for this tutorial, but typically are not

`$ bosh status`

* A message about SSL: Ubuntu backports openssl fixes; a Ruby gem bosh uses doesn't know that. See https://github.com/nahi/httpclient/blob/v2.7.1/lib/httpclient/ssl_config.rb#L450 for the context.

To remove warning message:

```
$ sudo sed -i '/RSA 1024 bit CA certificates are loaded due to old openssl compatibility/d' /var/lib/gems/2.0.0/gems/httpclient-2.7.1/lib/httpclient/ssl_config.rb
```



### Maria Shaldibina's "Learn BOSH"
  * [Learn BOSH](http://mariash.github.io/learn-bosh/)

clone the repo: `git clone https://github.com/mariash/learn-bosh-release`

Follow along in Learn BOSH, but with some modifications (as we're not targeting a remote bosh director).

Replace UUID in manifest. You can use an editor or this substitution:
```
sed -i "s/4796378f-cc91-4d93-a1b0-75a9af101708/$(bosh status --uuid)/" manifest.yml
```

```
ubuntu@agent-id-bosh-0:~/learn-bosh-release$ bosh status --uuid
190bb19e-2e13-4f69-99ec-316a5ac637cf
ubuntu@agent-id-bosh-0:~/learn-bosh-release$ grep uuid manifest.yml
director_uuid: 4796378f-cc91-4d93-a1b0-75a9af101708
ubuntu@agent-id-bosh-0:~/learn-bosh-release$ sed -i "s/4796378f-cc91-4d93-a1b0-75a9af101708/$(bosh status --uuid)/" manifest.yml
ubuntu@agent-id-bosh-0:~/learn-bosh-release$ grep uuid manifest.yml
director_uuid: 190bb19e-2e13-4f69-99ec-316a5ac637cf
ubuntu@agent-id-bosh-0:~/learn-bosh-release$
```

bosh-lite: only one VM. our “VMs” are containers.

```
bosh vms --details
bosh vms --vitals
bosh ssh

```

## Monit

`sudo su - && monit status`


## BOSH concepts


* BOSH release (versioned tarball of a software package, containing ALL of the assets that will be deployed - source files, configuration files, installation scripts, etc)

```
bosh releases
```

     * [bosh.io/releases](https://bosh.io/releases/)
     * 100+ community repos: https://github.com/cloudfoundry-community

* BOSH manifest - cluster config specification (in YAML) - what resources are going to be deployed, what services are going to be running on each of resources and properties that will be passed to services configuration files.
     * Sample manifests found in git repos for releases

* BOSH stemcell - A Stemcell is an operating system image (Ubuntu) plus BOSH specifics (agent, monit, etc) that BOSH uses to create VMs.

```
bosh stemcells
```

	* [Official stemcells on bosh.io](https://bosh.io/stemcells/)

Adding a bosh-lite stemcell to your bosh director:
`bosh upload stemcell https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-trusty-go_agent`

* BOSH deployment - a cluster of VMs based on the stemcell, running release software, defined by the manifest.

Looking at deployments: download manifest - see what’s actually in production!

```
bosh deployments
bosh download manifest <deployment-name> ~/deploymentname.yml
```


## Other releases

Redis:
http://bosh.io/releases/github.com/cloudfoundry-community/redis-boshrelease?all=1

`bosh upload release https://bosh.io/d/github.com/cloudfoundry-community/redis-boshrelease`

## Cloud Foundry

pre-installed AMI in us-east-1: ami-1bf91076

(fork of the provided bosh-lite AMIs)

Use `bosh start` and `bosh cck` to restart

## Recap

We talked about: AWS, Terraform, BOSH, Cloud Foundry
