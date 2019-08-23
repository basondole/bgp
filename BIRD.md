# The BIRD Internet Routing Daemon
The BIRD project aims to develop a fully functional dynamic IP routing daemon primarily targeted on (but not limited to) Linux, FreeBSD and other UNIX-like systems  
BIRD supports Internet Protocol version 4 and version 6 by running separate daemons. It establishes multiple routing tables, and uses BGP, RIP, and OSPF routing protocols, as well as statically defined routes.  
Learn more about the project via https://bird.network.cz/

# Installation and configuration on debian distributions  
Here we install bird on linux box running ubuntu 18 and configure local BGP using BIRD between the box and a junos router.

The precedding numbers on each step below correlate to the timestamps on the snapshots

* **1958:16** Install bird `sudo apt install bird`  
* **1958:21** Check the if ip forwarding is enable in the file `/etc/sysctl.conf` in this case the output lines are starting with `“#”` that means they are commented and hence ip forwarding is disabled  
* **1958:51** Use nano editor or any editor of your choice to edit the file `/etc/sysctl.conf` and remove the `“#”` preceeding the lines  
<pre>
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
</pre>

* **1958:51** Verify the lines have been uncommented and there is no preceeding `“#”`  
* **2013:31** Make a backup of the default bird configuration you may need it in the future

![installbird](https://user-images.githubusercontent.com/50369643/63584660-117a9500-c5a6-11e9-8698-2b115695253f.png)

* **2301:28** Edit the `/etc/bird/bird.conf` to specify your bgp and neighbor parameters and prefixes to announce.  
* **2301:33** Sample configuration I have put in the `/etc/bird/bird.conf`  
> I have used awk to only print the lines I have modified in the file

* **2301:36** Start the bird service on the machine. 
> If there is something wrong in the `/etc/bird/bird.conf` the service will not start. In such case the terminal will suggest some methods to follow so as to view the logs  

* **2301:48** Verify the service is started

![installbird2](https://user-images.githubusercontent.com/50369643/63585029-dcbb0d80-c5a6-11e9-9975-cd18b2475c95.png)

* **2315:06** Verify BGP by getting into the bird client issue command birdc as super user  
Use the command `show protocols all bgp1` to check the bgp summary. We see the configured neighbour 192.168.56.36 is up and we are receiving 1 prefix.
To view the prefix issue the command `show route protocol bgp1` This shows we are receiving a default route  
* **2315:54** Verify the host machine routing table. We see the default route we receive from the bgp neighbour 192.168.56.36 is installed in the local machine routing table with the next-hop 192.168.56.2  

![installbird3](https://user-images.githubusercontent.com/50369643/63585144-2277d600-c5a7-11e9-8932-6ee709a653bd.png)

* **2335:56** Login to the router 192.168.56.36 we see the bgp session is up. The router is advertising the default route to the machine running bird and receiving the two prefixes as we announce them on the bird configuration `@2301:33`

![installingbird4](https://user-images.githubusercontent.com/50369643/63585434-c82b4500-c5a7-11e9-982b-bf6ef58b9df6.png)
