+++
date = "2013-09-23T20:28:43-04:00"
draft = false
title = "LXC – Automating Containers aka Virtual Madness (Part 2)"

+++

Presenting Virtual Madness!
---------------------------

This is part 2 of our LXC blog post. If you missed the first half you can [read it here](https://jaysonmpaul.com/post/continuous-integration-at-etsy-1).

We already have much tooling around [making it easy to create a virtual machine](http://codeascraft.com/2012/03/13/making-it-virtually-easy-to-deploy-on-day-one/) for each developer, so it made sense to build our LXC virtualization tools into the same interface.

[![blog1](https://codeascraft.com/wp-content/uploads/2013/09/blog1-e1379952261153.png)](https://codeascraft.com/wp-content/uploads/2013/09/blog1-e1379952261153.png)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------

As shown above, our main page gives a quick look into all the LXC Containers that are running and the Jenkins instances to which they are attached. The containers are known colloquially as the Bobs, a reference to Bob the Builder. The interface also separates the Bobs by physical disk for ease of detecting if any of the virtual hosts' disks are overloaded.

Creation of a new Bob follows our one button deploy principle we strive for at Etsy:

[![blog2](https://codeascraft.com/wp-content/uploads/2013/09/blog2-e1379952393230.png)](https://codeascraft.com/wp-content/uploads/2013/09/blog2-e1379952393230.png)

The next numerically available hostname is automatically populated into the form by referencing all defined hosts in [Chef](http://www.opscode.com/chef/) and finding the first gap in numbering. The first physical host with enough free disk space is pre-selected from the drop-down and is determined according to the fact that we can fit 14 Bobs on each physical host (more on that later). Simply clicking the "Make it" button will kick off the process of creating an LXC container, and its progress is streamed via [websockets](http://dev.w3.org/html5/websockets/)to the browser in real-time.

The process of creating a new container is roughly as follows. Which of the disks on any given physical host has room for a new Bob is determined by a simple rule: the first disk holds a maximum of four Bobs, and the second and third disk hold a maximum of five each. Only four are allowed on the first disk in order to save room for the host OS and a base LVM volume which we clone for each Bob. The first numerically available IP address in any of our virtual host subnets is allocated to the Bob by iterating through reverse DNS lookups on the range until an NXDOMAIN is returned. [nsupdate](http://en.wikipedia.org/wiki/Nsupdate) is used to dynamically create the DNS entries for the new container. An empty LVM volume is then created by using lvcreate and mkfs.ext3, it is mounted, and rsync is used to clone the base volume to the empty volume.

The base volume is created in advance by cloning the physical host's root filesystem (which is configured almost exactly the same way as the Bobs) and then filtering out some unneeded items. /dev is fixed using [mknod](http://en.wikipedia.org/wiki/Mknod#Node_creation) because a simple rsync will render it useless. [Sed](http://en.wikipedia.org/wiki/Sed) is used to fix all the network settings in /etc/sysconfig. Unneeded services are disabled by creating a small bash script inside the volume and then [chrooting](http://en.wikipedia.org/wiki/Chroot) to run it.

Once the base volume is copied into our new volume, the Chef authorization key is removed so that the container is forced to automatically reregister with our Chef server as a new node with its newly specified hostname. Our default route to the gateway is added based on subnet and datacenter. An LXC config file is then created using the IP address and other options. lxc.cgroup.cpuset.cpus is set to 0 as our containers are not pinned to a specific set of CPU cores. This allows each executor to use as many CPU cores as can be allocated at any given time.

Finally, we bootstrap the node using Chef, bringing the node up to date with all our current packages and configurations. This is necessary because the base LVM volume from which the new container is cloned is created in advance and often has stale configurations.

There is also a batch UI which executes the exact same process as above but allows for the creation of Bobs in bulk:

[![blog3](https://codeascraft.com/wp-content/uploads/2013/09/blog3-e1379952479461.png)](https://codeascraft.com/wp-content/uploads/2013/09/blog3-e1379952479461.png)

Once Bobs are created, they are ready to be attached to Jenkins instances and given executor roles:

[![blog4](https://codeascraft.com/wp-content/uploads/2013/09/blog4-e1379952550551.png)](https://codeascraft.com/wp-content/uploads/2013/09/blog4-e1379952550551.png)

This interface allows for the choice of instance and label (executor), and the number of Bobs to attach. Unattached Bobs are surfaced by use of the [Jenkins API](https://wiki.jenkins-ci.org/display/JENKINS/Remote+access+API). After iterating through all unattached Bobs, taking into account the rule of only one heavy executor per disk, a [groovy](http://groovy.codehaus.org/) script is used to attach the Bob to Jenkins. The Jenkins API did not provide this functionality, but it does allow for the execution of groovy scripts, which we leveraged to automate this.

For our final bit of automation around this process, we created a one-button interface to wipe and rebuild the base LVM container on all physical hosts. This is necessary when the LVM containers fall very far out of date and start to cause issues bringing Bobs up to date using Chef. This interface simply performs the base container creation task described above across any number of physical hosts.

[![blog5](https://codeascraft.com/wp-content/uploads/2013/09/blog5-e1379952729482.png)](https://codeascraft.com/wp-content/uploads/2013/09/blog5-e1379952729482.png)

All in all, this virtual madness allowed us to turn our many manual processes of setting up LXC containers into simple one-button deploys. How do you handle this kind of stuff? Let us know in the comments!

Co-written by Jayson Paul, Patrick McDonnell and Nassim Kammah.

*You can follow Jayson on Twitter at [@jaysonmpaul](https://twitter.com/jaysonmpaul)\
You can follow Patrick on Twitter at [@mcdonnps](https://twitter.com/mcdonnps)*
