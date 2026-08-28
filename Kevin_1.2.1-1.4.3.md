# Overview of SD-WAN Technologies




Traditonal Wan: Hub and spoke topology with headquarters as the central point.


The cloud SHift: Application moved to AWS, Azure, and Google Cloud. 
Traffic Backhualing: all traffic routed through headquarters ( even cloud-bound traffic) causing latency and performance issues.
THe problem: why backhaul traffic to HQ when the destination is the cloud?


Transport Independence: SD-WAN allows for multiple transport options (MPLS, broadband, LTE) to be used simultaneously, providing flexibility and redundancy.

Overlay abstraction: Build logical toplogies on top of physical infrastructure

Direct Cloud Access:  Intelligent routing sends cloud traffic directly to cloud

Centralized Management:  Configure and monitor everything from one console.


Image: SD-WAN Architecture

![sdwan plane ](<images/sdwan.png>)





## SDWan deployment models



On-Premises: All SD-WAN components are deployed within the organization's own data centers and managed internally.
Cloud-Hosted: SD-WAN components are hosted in the cloud by a service provider, reducing the need for on-premises infrastructure and management.
Hybrid: Combines on-premises and cloud-hosted components, allowing organizations to leverage the benefits of both deployment models based on their specific needs and requirements.

Zero touch provisioning Workflow: SD-WAN devices can be automatically configured and provisioned without manual intervention, simplifying deployment and reducing operational overhead.






## SD-Access Fabric Architecture



manually Configured Access with ACLs


 Problem: if you give a person permission to access a server  and they move to another deparment or changed their ip they would then get denied


 SD access to try to move away from looking at your IP to looking at WHO you are.


The are able to achieve this thorugh Cisco ISE ( Identity Service engine )


ise will give your device a tag base on your mac address or something uniquily idenfiedable



LISP (Locator/ID Seperation Protocol )

A routing architecture that separates a device's identity from its network location to improve scalablity, mobility and traffic engineering 

EID ( Endpoint Identifier )

The IP address assigned to an endpoint device that identifies the device itself, independent of its location in the network 



RLoc ( Routing Locatoer)

The Ip address of a network device ( like a tunnel router ) that indicates where an endpoint is located within a network's topology

VxLan ( Virtual Extensible Lan)

An encapsulation protocol that creates Layer 2 overlay networks over Layer 3 infrasturcture

Underlay Network: the physcial devices

Overlay Network: the virtual connection between the device like (VXLan )




TO have SD ACCess network you NEED Cisco catalyst center otherwise it is just a fabric![SDACCESS](<images/sdaccess.png>)







![Roles](<images/Roles.png>)