README.md


# Step I - install Zookeeper
    * sudo apt-get update
    * sudo apt-get install zookeeper --> NOT needed
    * sudo apt-get install zookeeperd
    * telnet localhost 2181 --> type "ruok" + Enter to test installation
    * /usr/share/zookeeper/bin/zkCli.sh -server 127.0.0.1:2181  --> CLI

# Step II - install Nerve
    * mkdir -p /opt/smartstack/nerve
    * curl -sSL https://get.rvm.io | bash
    * source /home/vagrant/.rvm/scripts/rvm
    * rvm install 2.3
    * gem install nerve
    * create config file at /etc/nerve.conf.json
    * nerve -c /etc/nerve.conf.json > /var/log/nerve 2>&1 &

# Step III - install Synapse
    * Install RVM because Ruby sucks
    * curl -sSL https://get.rvm.io | bash
    * source /home/vagrant/.rvm/scripts/rvm
    * rvm install 2.3
    * gem install synapse
    * sudo apt-get install haproxy
