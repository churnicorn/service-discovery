README.md

# Introduction

This project stands up vm machines to act as each of he necessary components of smartstack.
More information on smartstack can be found here
* https://github.com/airbnb/smartstack-cookbook/blob/master/README.md
* https://github.com/airbnb/synapse
* http://jasonwilder.com/blog/2014/02/04/service-discovery-in-the-cloud/

# Step 1 - Bring up the VM machines
First, we need to install our machines. Under the root directory, if you do
```
    vagrant status
```
you will see six boxes listed. Each of these is defined in the VagrantFile.
Go ahead and bring each one up and running by opening up the six terminal windows
and running
```
    vagrant up {vm_name}
```
ex:
```
    vagrant up service_a_node_1
```

## Zookeeper
The zookeeper machines represent a zookeeper cluster, although we will initially start with just one in this setup.

## Service_A
There are two service_a nodes. This represents two machines running the same service. We will want to first see
a zookeeper instance interact with these two nodes and see how routing and registration/ deregistration
 behaves when one service goes down. Each service_a node will have a nerve and a synapse running on it, as well
 as a simple http service.

## Service_B
This is a machine representing a service different from service_a nodes. We have this to see the interaction
of one service calling another. Eventually, we will want service_b to call service_a and we will want to
see what that interaction looks like.

# Step 2 - Install all needed components on each of the machines
This part will end up in the lifesum-cli deployment scripts so it is a manual
step for now without much structure. Bear with us.

## Zookeeper
Let's go ahead and install zookeeper on each of the zookeeper machines
```
sudo apt-get update
sudo apt-get install zookeeperd
```
Then, let's see if zookeper is okay and if the installation worked.
This part is cute. Are you ready? We are about to ask zookeeper if it's okay.
First run this:
```
telnet localhost 2181
```
Then, ask the question:
```
ruok
```
When the command is run, zookeeper should respond with a 'iamok', but
they forgot to add a newline in the response so you'll have to really look
for it as it appears right before the next command line.

It'll look something like this:
```
vagrant@zookeeper1:~$ telnet localhost 2181
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
ruok
imokConnection closed by foreign host.
```

Go ahead and install and check the status on each of the zookeeper machines.

## Nerve And Synapse
Next, let's isntall nerve and synapse on each of the three service machines.

### Rails Virtual Envrionment
Download the rvm:
```
curl -sSL https://get.rvm.io | bash
```

Source and isntall the rvm:
```
source /home/vagrant/.rvm/scripts/rvm
rvm install 2.3
```

### Get The Gems
Get Nerve and Synapse gems:
```
gem install nerve
gem install synapse
```

## HAProxy
On each of the service machines, also go ahead and install HAProxy.
The load balancing is actually done on each of the machines/clients, and
NOT on the zookeeper machines. More on the overall explanation later.
```
sudo apt-get install haproxy
```

# Step 3 - Let's Do a Simple Setup Run : 1 zookeper, 1 service

## Zookeeper
Next, bring zookeeper up ONLY on the zookeeper1 machine. We will do this as well with the other machines later, but for now let's just work with one machine.
```
/usr/share/zookeeper/bin/zkCli.sh -server 127.0.0.1:2181
```

This will get you into zookeeper and it'll look like this:
```
vagrant@zookeeper1:~$ /usr/share/zookeeper/bin/zkCli.sh -server 127.0.0.1:2181
Connecting to 127.0.0.1:2181
Welcome to ZooKeeper!
JLine support is enabled

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: 127.0.0.1:2181(CONNECTED) 0]
```

Note that you can now list everything in zookeeper using `ls`:
```
[zk: 127.0.0.1:2181(CONNECTED) 0] ls
ZooKeeper -server host:port cmd args
    connect host:port
    get path [watch]
    ls path [watch]
    set path data [version]
    rmr path
    delquota [-n|-b] path
    quit
    printwatches on|off
    create [-s] [-e] path data acl
    stat path [watch]
    close
    ls2 path [watch]
    history
    listquota path
    setAcl path acl
    getAcl path
    sync path
    redo cmdno
    addauth scheme auth
    delete path [version]
    setquota -n|-b val path
[zk: 127.0.0.1:2181(CONNECTED) 1]
```

Nerve and Synapse are not a part of this list yet.

## Nerve, Synapse, HAProxy, and The Service
That's a mouthful of a title. There's a lot going on here - Zookeeper
doesn't actually do much at all - it is just a simple store of values.
All of the magic happens on each of the machines hosting the services which
also have Nerve and Synapse and HAProxy running on it.
Each machine is responsible for:

    * for registering itself with zookeeper - TODO - nerve or synapse?
    * continually sending its own health status to zookeeper to
        keep its registration in place - Nerve
    * finding the registered service it is attempting to call - Synapse
        * and utilizing HAProxy to load balance to the appropriate service



Do the next few steps ONLY on service_a_node_1
### Synapse
Synapse depends on a single config file in JSON format; it's usually called synapse.conf.json. The file has three main sections.

    * services: lists the services you'd like to connect.
    * haproxy: specifies how to configure and interact with HAProxy.
    * file_output (optional): specifies where to write service state to on the filesystem.

Go ahead and copy the syapse_service.conf contents from here into
/etc/synapse.conf.json

and run synapse :
```
synapse -c /etc/synapse.conf.json
```

This is also going to show the log output.

### Nerve
In another terminal window, let's run Nerve. Again, copy the nerve conf contents
from here to
/etc/nerve.conf.json

and run nerve:
```
nerve -c /etc/nerve.conf.json
```
This will again take up this terminal window with the output of nerve


### Status Check In
So far, we have

    * zookeeper1 running zookeeper,
    * service_a_node_1 running
        - Synapse
        - Node

The next step is to get HAProxy and a simple HTTP Service on service_a_node_1

### HAProxy

### Simple Service
Python allows you to do something super quick to get a service up and running.
Let's get into the tmp folder so we don't pollute our machine root dir too much:
```
cd /tmp/
```

Then, if you spin up a service, you should see failures:
```
python -m SimpleHTTPServer 8000
```
The failures are there because the nerve is configured to do a health check.
If you open up the nerve config again, you will see under 'checks' it is
attempting to serve a 'uri' under '/health'.
Our service currently knows nothing about health, so we can qucikly hack
fix this by just creating a file named 'health' in service_a_node_1. Stop the
service from running (ctrl+c) and :
```
touch health
```
Start up the service again and you should now see it reporting that it is healthy:
```
python -m SimpleHTTPServer 8000
```


# Step I - install Zookeeper
    * sudo apt-get update
    * sudo apt-get install zookeeper --> NOT needed
    * sudo apt-get install zookeeperd
    * telnet localhost 2181 --> type "ruok" + Enter to test installation
    * /usr/share/zookeeper/bin/zkCli.sh -server 127.0.0.1:2181  --> CLI

# Step II - install Nerve
    * mkdir -p /opt/smartstack/nerve -> SKIP
     curl -sSL https://get.rvm.io | bash
     source /home/vagrant/.rvm/scripts/rvm
     rvm install 2.3
     gem install nerve
    * create config file at /etc/nerve.conf.json
    * nerve -c /etc/nerve.conf.json > /var/log/nerve 2>&1 &

# Step III - install Synapse
    * Install RVM because Ruby sucks
    * curl -sSL https://get.rvm.io | bash
    * source /home/vagrant/.rvm/scripts/rvm
    * rvm install 2.3
    * gem install synapse
    * sudo apt-get install haproxy
