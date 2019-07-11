## Flow level Bandwidth Provisioning to the Docker Containers using OpenVSwitch (OVS)

The goal of this Proof of Concept to show the flow level bandwidth provisioning using the OvS Switch. 
We achieve it using below steps.
*    Docker Networking using OVS 
*    Creation of OVS Queues 
*    Adding the Flow Rules
*    Test the Bandwidth Provisioning 


In this setup we have two Docker Containers. We attach the two docker containers to OVS and test the bandwidth provisioning using them. 


![Alt text](./images/flowOVS.jpg?raw=true "OVS Setup")


### Dependencies for the setup
*    Ubuntu 16.04 
*    Docker

### Docker Networking using OvS

First of all we need to create the OVS based Docker Networking. 

*   Install OVS and ovs-docker utility
```
sudo su
apt-get -y install openvswitch-switch
cd /usr/bin
wget https://raw.githubusercontent.com/openvswitch/ovs/master/utilities/ovs-docker
chmod a+rwx ovs-docker
```
*  Create the OVS bridge
```
ovs-vsctl add-br ovs-br1
ovs-vsctl show
```
*  Configure the OVS-br and create two ubuntu Docker Containers without network
```
ifconfig ovs-br1 10.0.0.1 netmask 255.255.255.0 up
docker run -ti -d --rm --name cont1 --cap-add NET_ADMIN --net=none -v ~/multi_threaded_iperf/:/code btvk/delay_ns
docker run -ti -d --rm --name cont2 --cap-add NET_ADMIN --net=none -v ~/multi_threaded_iperf/:/code btvk/delay_ns
```

*  Connect the containers to OVS bridge 
```
sudo ovs-docker add-port ovs-br1 eth0 cont1 --ipaddress=10.0.0.2/24 --gateway=10.0.0.1
sudo ovs-docker add-port ovs-br1 eth0 cont2 --ipaddress=10.0.0.3/24 --gateway=10.0.0.1
```

*  Add NAT Rules (Change your pubintf from "enp0s1" appropriatly)
```
export pubintf=enp0s1
export privateintf=ovs-br1
iptables -t nat -A POSTROUTING -o $pubintf -j MASQUERADE
iptables -A FORWARD -i $privateintf -j ACCEPT
iptables -A FORWARD -i $privateintf -o $pubintf -m state --state RELATED,ESTABLISHED -j ACCEPT
```

*  Test the connection between two containers connected via OVS bridge using Ping command
```
#log in to cont1 docker 
docker exec -it cont1 bash
#10.0.0.3 is IP Addr. of cont2  
ping 10.0.0.3
```


### Creation of OVS Queues

* We need the OVS port-ids of **cont1** and **cont2** that are connected to **ovs-br1** (See the above figure). 
```
#cont1 is the container name
#eth0 is the ovs-docker interface in the container
ovs-vsctl --data=bare --no-heading --columns=name find interface \
             external_ids:container_id="cont1"  \
             external_ids:container_iface="eth0"
#output from above will the ovs-port-id for cont1. Note it for next step.  

ovs-vsctl --data=bare --no-heading --columns=name find interface \
             external_ids:container_id="cont2"  \
             external_ids:container_iface="eth0"
#output from above will the ovs-port-id for cont2. Note it for next step.             
```
* Let say we got the below values for ovs-port-ids (from the above step).
```
cont1 ovs-br1 e0715799fb694_l
cont2 ovs-br1 38c9206b60f94_l
```

* Now, we create OvS Queues at **ovs-br1**. We create two queues at _e0715799fb694_l_ and _38c9206b60f94_l_ with maximum rates of 30Mbps in one queue and 5Mbps in other queue.

```
ovs-vsctl set port e0715799fb694_l qos=@qos1 -- --id=@qos1 create qos type=linux-htb \
    queues:0=@queue0 queues:1=@queue1 -- \
    --id=@queue0 create queue other-config:max-rate=30000000 -- \
    --id=@queue1 create queue other-config:max-rate=5000000

ovs-vsctl set port 38c9206b60f94_l qos=@qos2 -- --id=@qos2 create qos type=linux-htb \
    queues:2=@queue2 queues:3=@queue3 -- \
    --id=@queue2 create queue other-config:max-rate=30000000 -- \
    --id=@queue3 create queue other-config:max-rate=5000000
```


### Adding the Flow Rules
* To list QoS rules
```
ovs-vsctl list qos
```

* Use below commands to create flow rules for TCP/UDP ports. If the TCP destination port is 9090, we set the packet to go through queue-0 and if the TCP destination port is 9091, we set the packet to go through queue-1. Else, it goes through the default queue.

```
ovs-ofctl add-flow ovs-br1 priority=65535,tcp,tcp_dst=9090,actions=set_queue:0,normal
ovs-ofctl add-flow ovs-br1 priority=65535,tcp,tcp_dst=9091,actions=set_queue:1,normal
ovs-ofctl add-flow ovs-br1 priority=0,actions=normal
```

* To view all added flow rules in a switch
```
ovs-ofctl dump-flows ovs-br1
```
* To delete all flow rules in a switch
```
ovs-ofctl del-flows ovs-br1
```

### Test the Bandwidth Provisioning
* Execute the **cont2**

```
docker exec -it cont2 bash
apt install -y iperf3
iperf3 -s -p 9090
```

* Parallely in another terminal execute **cont2**
```
docker exec -it cont1 bash
apt install -y iperf3
iperf3 -c 10.0.0.3 -p 9090
```

* Observe the iperf3 sender throughput at **cont1** after  iperf session. It should be nearly 30 Mbps. If we repeat the above steps by changing the iperf3 port to 9091, we should see iperf3 sender throughput at **cont1** after  iperf session to be nearly 5Mbps.



* **References**
    * https://developer.ibm.com/recipes/tutorials/using-ovs-bridge-for-docker-networking/
    * http://docs.openvswitch.org/en/latest/faq/qos/