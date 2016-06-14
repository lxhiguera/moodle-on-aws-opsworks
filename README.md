# Run Moodle on Amazon AWS Opsworks (Chef 12 Linux)

Attempting to setup Moodle server on Opsworks
- Apache 2.4
- PHP 5.6
- App install from Github

## Setup

### Pre-req

#### RDS (Database)

AWS->Services->RDS

Set up a t2.micro or t2.small RDS instance.
- Step 1: MySQL
- Step 2: Production MySQL or Dev (up to you)
- Step 3: 
-- DB Engine Version 5.7.latest
-- DB Instance Class: t2.micro will do for a small setup
-- Multi-AZ: Up to you
-- Storage type: SSD either
-- Allocated storage: At least 50-100GB
-- DB instance identifier etc: "Moodle" / Up to you, but note the details you enter
- Step 4:
-- VPC: Same as all others in this setup
-- Security group: moodle-opsworks-all
-- Parameter group: "Moodle"

Note: If you used triggers in your Moodle database, you will need to create an RDS 'parameter group' and set your RDS instance to use it
- Parameter groups -> new 
-- Family: mysql5.7
-- Name: "Moodle"
-- Edit parameters
-- log_bin_trust_function_creators => 1

#### ELB (Load balancer)

AWS->Services->EC2->Load balancers

Create load balancer
- Load balancer name: Moodle
- Listener config:
-- HTTP80->HTTP80
-- HTTPS443->HTTP80
- Security groups:
-- Default VPC security group (?)
-- moodle-opsworks-all
-- moodle-opsworks-webserver
- Security settings
-- Either upload your SSL cert, or create one with ACM - it's easy
- Health check:
-- Ping path: /aws-up-check.php
- Instances: none yet


#### EC2 Security Groups: 

Create the following security groups in your target region:

- moodle-opsworks-all
-- ssh: from: your IP
-- all traffic: from security group: moodle-opsworks-all (this SG)

- moodle-opsworks-webserver
-- http: from 0.0.0.0
-- https: from 0.0.0.0


### Opsworks 

#### Stack

- Default operating system: Amazon Linux [latest]
- Default SSH key: [put in one of your EC2 SSH keys here, or you'll regret it when you go to troubleshoot]
- Chef version: 12
- Use custom Chef cookbooks: yes
- Repo: https://github.com/jamesoflol/opsworks-demo.git
- Use OpsWorks security groups: no

#### Layer: moodle-web-server

Security:
- moodle-opsworks-all
- moodle-opsworks-webserver

Recipes:
- Configure: moodle_web_server::configure
- Deploy: moodle_web_server::deploy

Network:
- Elastic load balancer: Moodle
- Public IP Address: Yes

#### Layer: moodle-data-server

Security:
- moodle-opsworks-all

Recipes:
- Configure: moodle_data_server

Network:
- Public IP Address: Yes

EBS Volumes:
- Mount point: /vol/moodledata
- Size total: 100GB (or whatever's approps)
- Volume type: General Purpose SSD

#### Layer: memcached

Security:
- moodle-opsworks-all

Recipes:
- Configure: memcached

Network:
- Public IP Address: Yes

### Layer: RDS

- Instance/User/Password: [as specified when setting up RDS above]



## Todo:

high:
- elb
- get it more working...

med:
- s3 backup/restore
- cloudformation script for all this

low:
- moodle_web_server: fix deploy script so that it doesn't need to symlink /var/www/html

## Notes for playing around with Chef local in SSH on individual machines

sudo mkdir /var/chef/cookbooks/test
sudo mkdir /var/chef/cookbooks/test/recipes
sudo echo "" > /var/chef/cookbooks/test/recipes/default.rb
sudo nano /var/chef/cookbooks/test/recipes/default.rb
--put some chef script in now--
sudo chef-client -z -o test
