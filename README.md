AWS CloudFormation Templates


# Introduction

This repo contains various AWS cloudformation templates for creating the following database / messaging system clusters: -

1. MongoDB Production Cluster
2. Elasticsearch Cluster
3. RabbitMQ Cluster

**NOTE:** These templates assume that the following is in place: -

1. Security Groups created for each cluster.
2. VPC created - these will all be installed in the private subnet.
3. Route53 private zone created (recommended best practices).
4. AWS IAM role created called MonitorRole - this is for installing CloudWatch metrics such as memory / diskspace monitoring.

Hopefully you find these templates useful - feel free to edit, remove and add sections depending on your use-case.

# Download

To download run the following command: -

    git clone https://github.com/adoreboard/aws-cloudformation-templates.git


# Further Reading

This is a very good question and I went through this very painful journey myself recently. I am writing a fairly extensive answer here in the hope that some of these thoughts of running a MongoDB cluster via CloudFormation are useful to others.

I'm assuming that you're creating a MongoDB production cluster as follows: -

3 config servers (micros/smalls instances can work here)
At least 1 shard consisting of e.g. 2 (primary & secondary) shard instances (minimum or large) with large disks configured for data / log / journal disks.
arbiter machine for voting (micro probably OK).
i.e. https://docs.mongodb.org/manual/core/sharded-cluster-architectures-production/

Like yourself, I initially tried the AWS MongoDB CloudFormation template that you posted in the link (https://s3.amazonaws.com/quickstart-reference/mongodb/latest/templates/MongoDB-VPC.template) but to be honest it was far, far too complex i.e. it's 9,300 lines long and sets up multiple servers (i.e. replica shards, configs, arbitors, etc). Running the CloudFormation template took ages and it kept failing (e.g. after 15 mintues) which meant the servers all terminated again and I had to try again which was really frustrating / time consuming.

The solution I went for in the end (which I'm super happy with) was to create separate templates for each type of MongoDB server in the cluster e.g.

MongoDbConfigServer.template (template to create config servers - run this 3 times)
MongoDbShardedReplicaServer.template (template to create replica - run 2 times for each shard)
MongoDbArbiterServer.template (template to create arbiter - run once for each shard)
NOTE: templates available at https://github.com/adoreboard/aws-cloudformation-templates

The idea then is to bring up each server in the cluster individually i.e. 3 config servers, 2 sharded replica servers (for 1 shard) and an arbitor. You can then add custom parameters into each of the templates e.g. the parameters for the replica server could include: -

InstanceType e.g. t2.micro
ReplicaSetName e.g. s1r (shard 1 replica)
ReplicaSetNumber e.g. 2 (used with ReplicaSetName to create name e.g. name becomes s1r2)
VpcId e.g. vpc-e4ad2b25 (not a real VPC obviously!)
SubnetId e.g. subnet-2d39a157 (not a real subnet obviously!)
GroupId (name of existing MongoDB group Id)
Route53 (boolean to add a record to an internal DNS - best practices)
Route53HostedZone (if boolean is true then ID of internal DNS using Route53)
The really cool thing about CloudFormation is that these custom parameters can have (a) a useful description for people running it, (b) special types (e.g. when running creates a prefiltered combobox so mistakes are harder to make) and (c) default values. Here's an example: -

    "Route53HostedZone": {
        "Description": "Route 53 hosted zone for updating internal DNS (Only applicable if the parameter [ UpdateRoute53 ] = \"true\"",
        "Type": "AWS::Route53::HostedZone::Id",
        "Default": "YA3VWJWIX3FDC"
    },
This makes running the CloudFormation template an absolute breeze as a lot of the time we can rely on the default values and only tweak a couple of things depending on the server instance we're creating (or replacing).

As well as parameters, each of the 3 templates mentioned earlier have a "Resources" section which creates the instance. We can do cool things via the "AWS::CloudFormation::Init" section also. e.g.

"Resources": {

    "MongoDbConfigServer": {
        "Type": "AWS::EC2::Instance",
        "Metadata": {
            "AWS::CloudFormation::Init": {
                "configSets" : {
                    "Install" : [ "Metric-Uploading-Config", "Install-MongoDB", "Update-Route53" ]
                },
The "configSets" in the previous example shows that creating a MongoDB server isn't simply a matter of creating an AWS instance and installing MongoDB on it but also we can (a) install CloudWatch disk / memory metrics (b) Update Route53 DNS etc. The idea is you want to automate things like DNS / Monitoring etc as much as possible.

IMO, creating a template, and therefore a stack for each server has the very nice advantage of being able to replace a server extremely quickly via the CloudFormation web console. Also, because we have a server-per-template it's easy to build the MongoDB cluster up bit by bit.

My final bit of advice on creating the templates would be to copy what works for you from other GitHub MongoDB CloudFormation templates e.g. I used the following to create the replica servers to use RAID10 (instead of the massively more expensive AWS provisioned IOPS disks).

https://github.com/CaptainCodeman/mongo-aws-vpc/blob/master/src/templates/mongo-master.template

In your question you mentioned auto-scaling - my preference would be to add a shard / replace a broken instance manually (auto-scaling makes sense with web containers e.g. Tomcat / Apache but a MongoDB cluster should really grow slowly over time). However, monitoring is very important, especially the disk sizes on the shard servers to alert you when disks are filling up (so you can either add a new shard to delete data). Monitoring can be achieved fairly easily using AWS CloudWatch metrics / alarms or using the MongoDB MMS service.

If a node goes down e.g one of the replicas in a shard, then you can simply kill the server, recreate it using your CloudFormation template and the disks will sync across automatically. This is my normal flow if an instance goes down and generally no re-configuration is necessary. I've wasted far too many hours in the past trying to fix servers - sometimes lucky / sometimes not. My backup strategy now is run a mongodump of the important collections of the database once a day via a crontab, zip up and upload to AWS S3. This means if the nuclear option happens (complete database corruption) we can recreate the entire database and mongorestore in an hour or 2.

However, if you create a new shard (because you're running out of space) configuration is necessary. For example, if you are adding a new Shard 3 you would create 2 replica nodes (e.g. primary with name => mongo-s3r1 / secondary with name => mongo-s3r2) and 1 arbitor (e.g. with name mongo-s3r-arb) then you'd connect via a MongoDB shell to a mongos (MongoDB router) and run this command: -

sh.addShard("s3r/mongo-s3r1.internal.mycompany.com:27017,mongo-s3r2.internal.mycompany.com:27017")
NOTE: - This commands assumes you are using private DNS via Route53 (best practice). You can simply use the private IPs of the 2 replicas in the addShard command but I have been very badly burned with this in the past (e.g. serveral months back all the AWS instances were restarted and new private IPs generated for all of them. Fixing the MongoDB cluster took me 2 days as I had to reconfigure everything manually - whereas changing the IPs in Route53 takes a few seconds ... ;-)

You could argue we should also add the addShard command to another CloudFormation template but IMO this adds unnecessary complexity because it has to know about a server which has a MongoDB router (mongos) and connect to that to run the addShard command. Therefore I simply run this after the instances in a new MongoDB shard have been created.

Anyways, that's my rather rambling thoughts on the matter. The main thing is that once you have the templates in place your life becomes much easier and defo worth the effort! Best of luck! :-)
