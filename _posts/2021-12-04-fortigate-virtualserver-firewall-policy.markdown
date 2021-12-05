We are in the midst of migrating from our old Fortigate 4 series to a new Fortigate Firewall appliance using FortiOS v5.6. 

The only showstopper I ran into so far was the Virtual Server definitions for our Kubernetes cluster and Docker Swarm. We have 9 of these set up for various port redirects on 80 and 443 that Traefik handles and sends on to the correct internal ports. 

We got everything set up: Virtual Server definitions (The IP and Port that the public contacts), Real Server definitions (The IP and Port that the cluster nodes are listening on), Health Checks (Which cluster nodes are currently acccepting traffic), 
and the firewall rule (It is okay to allow through web requests from the internet to this IP and port) to tie it all together. The Firewall rule was accepting incoming traffic on 80 and 443 to the Virtual Server VIPs. 

However, once we started testing, I could not get it to go all the way through to the cluster nodes. We kept getting log messages that looked like this: 

Dec  1 14:43:00 FW-appliance date=2021-12-01 time=14:42:59 devname="FW" type="traffic" subtype="forward" level="notice" vd="DOCKER" eventtime=1638387779 srcip=MYWORKSTATIONIP srcport=65167 dstip=VIRTUALSERVERVIP dstport=80 proto=6 action="deny" policyid=0 policytype="policy" service="HTTP" dstcountry="United States" srccountry="United States" trandisp="dnat" tranip=CLUSTERNODEIP tranport=7445 duration=0 sentbyte=0 rcvdbyte=0 sentpkt=0 

This part stands out: __action="deny" policyid=0__. policyid=0 is the Implicit Deny, which is what the firewall does when it has not found any policies that match the observed traffic. This means that our attempts to reach port 80 on VIRTUALSERVERVIP are not matching the rule that I created that is supposed to accept them. 

I tried searching around the internet for log entries that look like this, with the trandisp="dnat" tranip=CLUSTERNODEIP tranport=CLUSTERPORT and did not find a lot of useful results. I ended up on Fortinet's documentation page. In their docs, they specify that you need to have NAT enabled for the Firewall Policy that is accepting the traffic for the Virtual Server, with Use Outgoing Interface Address selected as the IP Pool Configuration. 
I tried enabling NAT on our policy, but that did not help. 

I spoke to some of our Network Engineering and Network Security folks and together we tried several variations on the existing policy. Eventually I ended up getting the correct information from one of the more senior Security folks. Apparently, he had run across a similar problem a couple of years earlier. Here is the advice he sent to me: 

* You cannot have multiple VIPs on the same rule. This worked on the old firewall, but changed in the newer versions of FortiOS. 
* You need to have both the destination and translated ports allowed in the same policy, because it is going from the destination port on the interface to the translated port on the interface to the servers. 
(This is the part of the deny log message that mentions trandisp tranip tranport, all of the pieces of translation need to be allowed.) 

I followed his advice and it works now! I'm hoping that this info will help someone else in a similar situation, since our next step was going to be opening a support ticket with FortiNet, with substantial violence to our project timeline. 