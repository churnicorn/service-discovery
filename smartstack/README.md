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

Source and install the rvm:
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
sudo /usr/share/zookeeper/bin/zkServer.sh start
```

Use the zookeeper cli to look inside of zookeeper:
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
This will again take up this terminal window with the output of nerve.
It will tell you that it's up and running and then it will proceed to complain
that it can't connect to zookeeper. That's okay. Leave it as is.


### Status Check In
So far, we have

    * zookeeper1 running zookeeper,
    * service_a_node_1 running
        - Synapse
        - Node

The next step is to get HAProxy and a simple HTTP Service on service_a_node_1

### HAProxy

#### Extras/ Debugging
If you ever need to check the configuration:
```
cat /etc/default/haproxy
```
If you need to relaod HAProxy:
```
service haproxy reload
```

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
They look like this:
```
192.168.11.102 - - [22/Jun/2016 09:10:42] code 404, message File not found
192.168.11.102 - - [22/Jun/2016 09:10:42] "GET /health HTTP/1.1" 404 -
```
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


### Status Check - Again
A lot of things all of a sudden startd working.
We now have a zookeeper with a service host that is running synapse, a nerve, and a service. Here's what should have happened in each of the 4 terminal windows:

##### The Service
You'll see successful gets for health in the service terminal window.

##### Nerve
Back, in the 'Nerve' terminal window, you will see an update. It stops complaining and let's us know the servic eis actually up:
```
[2016-06-22T09:11:14.175835 #32025]  INFO -- Nerve::ServiceCheck::HttpServiceCheck: nerve: service check http-192.168.11.102:8000/health transitions to up after 3 successes
I, [2016-06-22T09:11:14.198057 #32025]  INFO -- Nerve::ServiceWatcher: nerve: service simple_http_service is now up
```

##### Synapse
Synapse shows that it's starting. ZooKeeper watcher is starting on
'''
simple_http_service @ hosts: 192.168.11.101:2181, path: /nerve/services/simple_http_service/services
'''
We then see

    * the Zookeeper Watcher connects to the service
    * it creates a pool connection
    * it tells us a successful connection was then made
    * lets us know it's discovering backends for simple_http_service
    * Synapse configures HAProxy
    * We keep trying to discover backends and at some point one (our lone service) is discoevred:
        - ` discovered 1 backends for service simple_http_service `
        - After discovery, it configures HAProxy again


# Step 4 - Debugging - TODO - move to WIKI Page
At this point, we have the minimal components all up and running.

## Service Crashed
Just stop the service with 'ctrl + c'

#### Service
Well.. it's down. Nothing to see here. You'll be back at the /tmp folder.

#### Nerve
If the service crashes, 'Nerve' will let us know it received an error from our service. We configured the nerve to see our service as unhealthy after 3 errors, so after it tells us it's getting the errors 3 times, we will get a connection failure message every 2 seconds or so.

#### Synapse
Interestingly, Synapse reports with a warning that there are no backends for our service now and it'll just use the last known backend.
TODO - does this mean that if service_b_node_1 tried to ping servoce
, it will still think it is healthy?
```
WARN -- Synapse::ServiceWatcher::ZookeeperWatcher: synapse: no backends for service simple_http_service and no default servers for service simple_http_service; using previous backends: [{"name"=>"service_a_node_1", "host"=>"192.168.11.102", "port"=>8000, "id"=>1, "weight"=>nil, "haproxy_server_options"=>nil, "labels"=>nil}]
```

#### Zookeeper
Nothing new to see here

#### Recovery
spin up the service again and Nerve will say happily it is now healthy and up after three successful health checks. Synapse again discovers backends and reconfigures the HAProxy config. The service terminal window will keep getting a 200.

## Zookeeper Crashed - TODO - how do we crash it?
Stop ZooKeeper with:
```
sudo /usr/share/zookeeper/bin/zkServer.sh stop
```

## Synapse Crashed - TODO
None of the terminal windows display anything, BUT if synapse is down - can
service_a_node_1 still make calls to other services?

#### Recovery
Bringing Synapse back up shows that it reconfigures and joins our ecosystem.
Just as before, Synapse tells us it has started the zookeeper watcher and that
it found one backend for our service.

## Nerve Crashed

#### Nerve
It tells us that

    * Nerve - reaping all watchers
    * ServiceWatcher - it's ending service watch simple_http_service
    * Reporter:Zookeeper - it's removing zk node at /nerve/services/simple_http_service/services/service_a_node_1_0000000002
    * Reporter:Zookeeper - it's closing the connection

#### Synapse
Synapse uses the last known backend and understands it's not getting a report from Nerve anymore. The same thign happened when Nerve was up but the service crashed.
```
Synapse::ServiceWatcher::ZookeeperWatcher: synapse: no backends for service simple_http_service and no default servers for service simple_http_service; using previous backends: [{"name"=>"service_a_node_1", "host"=>"192.168.11.102", "port"=>8000, "id"=>2, "weight"=>nil, "haproxy_server_options"=>nil, "labels"=>nil}]
```
#### Zookeeper
Nothing interesting here

#### The Service
Still running happily, but with 200 health checks halted.
With Nerve down, nothing is requesting the health check from our service.

#### Recovery
Nerve comes up, let's us know the initial healtch check returned true, and starts reporting that our service is healthy as it keeps pinging it. The service now starts spitting 200s back on the screen and Synapse rediscovered our sole backend and updated the HAProxy config.


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
