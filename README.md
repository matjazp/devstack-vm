# Neutron-enabled DevStack in a Vagrant VM with Ansible

This repository contains a Vagrantfile and an accompanying Ansible playbook
that sets up a VirtualBox virtual machine that installs [DevStack][4].

You'll also be able to ssh directly from your laptop into the VMs without
needing to the ssh into the Vagrant box first.

Ansible generates a `local.conf` file that defaults to:

 * Use Neutron for networking
 * Disable security groups
 * No Application Catalog

You can enable Swift, Heat, Application Catalog and security groups
by editing the devstack.yml file.

This project was inspired by Brian Waldon's [vagrant_devstack][1] repository.

## Memory usage

By default, the VM uses 6GB of RAM and 2 cpus. If you want to change this, edit the
following lines in Vagrantfile:

        vb.customize ["modifyvm", :id, "--memory", 6144]
        vb.customize ["modifyvm", :id, "--cpus", 2]

## Prereqs


Install the following applications on your local machine first:

 * [VirtualBox][5]
 * [Vagrant][2]
 * [Ansible][3]

If you want to try out the OpenStack command-line tools once DevStack is
running, you'll also need to install the following Python packages:

  * python-novaclient
  * python-neutronclient
  * python-openstackclient

The easiest way to install Ansible and the Python packages are with pip:

    sudo pip install -r requirements.txt

## Boot the virtual machine and install DevStack

Grab this repo and do a `vagrant up`, like so:

    git clone https://github.com/lorin/devstack-vm
    cd devstack-vm
    vagrant up

The `vagrant up` command will:

 1. Download an Ubuntu 14.04 (trusty) vagrant box if it hasn't previously been downloaded to your machine.
 2. Boot the virtual machine (VM).
 3. Clone the DevStack git repository inside of the VM.
 4. Run DevStack inside of the VM.
 5. Add eth2 to the br-ex bridge inside of the VM to enable floating IP access from the host machine.

It will take at least ten minutes for this to run, and possibly much longer depending on your internet connection and whether it needs to download the Ubuntu vagrant box.



## Troubleshooting

### Fails to connect

You may ocassionally see the following error message:

```
[default] Waiting for VM to boot. This can take a few minutes.
[default] Failed to connect to VM!
Failed to connect to VM via SSH. Please verify the VM successfully booted
by looking at the VirtualBox GUI.
```

If you see this, retry by doing:

    vagrant destroy --force && vagrant up


## Logging in the virtual machine

The VM is accessible at 192.168.27.100

You can type `vagrant ssh` to start an ssh session.

Note that you do not need to be logged in to the VM to run commands against the OpenStack endpoint.


## Loading OpenStack credentials

From your local machine, to run as the demo user:

    source demo.openrc

To run as the admin user:

    source admin.openrc

## Horizon

* URL: http://192.168.27.100
* Username: admin or demo
* Password: password


## Initial networking configuration

![Network topology](topology.png)


DevStack configures an internal network ("private") and an external network ("public"), with a router ("router1") connecting the two together. The router is configured to use its interface on the "public" network as the gateway.

```
$ openstack network list
+--------------------------------------+---------+------------------------------------------------------------------------+
| ID                                   | Name    | Subnets                                                                |
+--------------------------------------+---------+------------------------------------------------------------------------+
| 3d910901-12a0-4997-8335-948c66e1ab46 | public  | 1c458c90-3bd3-45b1-a9bf-6ed8cd56e128,                                  |
|                                      |         | 94f2f87c-c8a4-48e5-a27c-752e7be14988                                   |
| c83dc6a9-615e-4a42-b462-b5d9871a923f | private | 6e58ab8b-bc1a-4ae8-9233-f2d69a5c1821,                                  |
|                                      |         | 830a36ce-4bb4-4266-8411-5d4447e8e2e3                                   |
+--------------------------------------+---------+------------------------------------------------------------------------+

$ neutron router-list
+--------------------------------------+---------+------------------------------------------------------------------------+
| id                                   | name    | external_gateway_info                                                  |
+--------------------------------------+---------+------------------------------------------------------------------------+
| c182627f-2c78-4f0e-aa14-f740aa7a02d3 | router1 | {"network_id": "3d910901-12a0-4997-8335-948c66e1ab46",                 |
|                                      |         | "external_fixed_ips": [{"ip_address": "172.24.4.2", "subnet_id":       |
|                                      |         | "1c458c90-3bd3-45b1-a9bf-6ed8cd56e128"}, {"ip_address": "2001:db8::1", |
|                                      |         | "subnet_id": "94f2f87c-c8a4-48e5-a27c-752e7be14988"}], "enable_snat":  |
|                                      |         | true}                                                                  |
+--------------------------------------+---------+------------------------------------------------------------------------+
```


## Add ssh and ping to the default security group

    openstack security group rule create default --proto tcp --dst-port 22
    openstack security group rule create default --proto icmp

	
## Launch a cirros instance and attach a floating IP.

Source the credentials of the "demo" user and boot an instance.

    source demo.openrc
    nova keypair-add --pub-key ~/.ssh/id_rsa.pub mykey
    nova boot --flavor m1.tiny --image cirros-0.3.4-x86_64-uec --key-name mykey cirros

Once the instance has booted, get its ID.

    $ nova list

    +--------------------------------------+--------+--------+------------+-------------+------------------------------------------------------+
    | ID                                   | Name   | Status | Task State | Power State | Networks                                             |
    +--------------------------------------+--------+--------+------------+-------------+------------------------------------------------------+
    | 62cf0635-aa9e-4223-bbcd-3808966959c1 | cirros | ACTIVE | -          | Running     | private=fdbc:59ac:894:0:f816:3eff:fefe:221, 10.0.0.3 |
    +--------------------------------------+--------+--------+------------+-------------+------------------------------------------------------+

Use the instance ID to get its neutron port :

    $ neutron port-list -c id --device_id b24fc4ad-2d66-4f28-928b-f1cf78075d33

    +--------------------------------------+
    | id                                   |
    +--------------------------------------+
    | 02491b08-919e-4582-9eb7-f8119c03b8f9 |
    +--------------------------------------+


Use the neutron port ID to create an attach a floating IP to the "public"" network:

    $ neutron floatingip-create public --port-id 02491b08-919e-4582-9eb7-f8119c03b8f9

    Created a new floatingip:
    +---------------------+--------------------------------------+
    | Field               | Value                                |
    +---------------------+--------------------------------------+
    | fixed_ip_address    | 10.0.0.3                             |
    | floating_ip_address | 172.24.4.227                         |
    | floating_network_id | 5770a693-cfc7-431d-ae29-76f36a2e63c0 |
    | id                  | 480524e1-a5b3-491f-a6ee-9356fc52f81d |
    | port_id             | 02491b08-919e-4582-9eb7-f8119c03b8f9 |
    | router_id           | 0deb0811-78b0-415c-9464-f05d278e9e3d |
    | tenant_id           | 512e45b937a149d283718ffcfc36b8c7     |
    +---------------------+--------------------------------------+

Finally, access your instance:

    ssh cirros@172.24.4.227


## Python bindings example

The included `boot-cirros.py` file illustrates how to execute all of the
above commands using the Python bindings.

## Allow VMs to connect out to the Internet

By default, VMs started by OpenStack will not be able to connect to the
internet. For this to work, your host machine must be configured to do NAT
(Network Address Translation) for the VMs.

### On Mac OS X

#### Enable IP forwarding

Turn on IP forwarding if it isn't on yet:

    sudo sysctl -w net.inet.ip.forwarding=1

Note that you have to do this each time you reboot.

#### Edit the pfctl config file to NAT the floating IP subnet

Edit `/etc/pf.conf` as root, and add the following line after the "net-anchor" line:

    nat on en0 from 172.24.4.1/24 -> (en0)

#### Load the file and enable PF

    sudo pfctl -f /etc/pf.conf
    sudo pfctl -e

(From [Martin Nash's blog][6]. See info there on how to make the IP forwarding
persist across reboots ).


### On Linux

To enable NAT, issue the following commands in your host, as root:

```
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

[1]: https://github.com/bcwaldon/vagrant_devstack
[2]: http://vagrantup.com
[3]: http://ansible.com
[4]: http://devstack.org
[5]: http://virtualbox.org
[6]: http://blog.nasmart.me/internet-access-with-virtualbox-host-only-networks-on-os-x-mavericks/

## Troubleshooting

Logs are in `/opt/stack/logs`

### Instance immediately goes into error state

Check the nova-conductor log and search for ERROR

```
vagrant ssh
less -R /opt/stack/logs/n-cond.log
```

For example, if it's failing because there isn't enough free memory in the
virtual machine, you'll see an error like this:

```
2016-08-01 05:42:50.237 ERROR nova.scheduler.utils [req-581add06-ba33-4b5d-9a1b-af7c74f3ce86 demo demo] [instance: 70713d2f-96fa-4ee7-a73a-4e019b78b1f9] Error from last host: vagrant-ubuntu-trusty-64 (node vagrant-ubuntu-trusty-64): [u'Traceback (most recent call last):\n', u'  File "/opt/stack/nova/nova/compute/manager.py", line 1926, in _do_build_and_run_instance\n    filter_properties)\n', u'  File "/opt/stack/nova/nova/compute/manager.py", line 2116, in _build_and_run_instance\n    instance_uuid=instance.uuid, reason=six.text_type(e))\n', u"RescheduledException: Build of instance 70713d2f-96fa-4ee7-a73a-4e019b78b1f9 was re-scheduled: internal error: process exited while connecting to monitor: Cannot set up guest memory 'pc.ram': Cannot allocate memory\n\n"]
```






