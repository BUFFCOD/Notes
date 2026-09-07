## Hypervisor Type 1 and 2


Hypervisors are software, firmware, or hardware that create and run virtual machines (VMs). A computer on which a hypervisor runs one or more VMs is called a host machine, and each VM is called a guest machine. The hypervisor presents the guest operating systems with a virtual operating platform and manages the execution of the guest operating systems.



Type 1 (native or bare-metal) hypervisors run directly on the host's hardware to control the hardware and to manage guest operating systems. For example, VMware ESXi, Microsoft Hyper-V, and Xen are Type 1 hypervisors.

Type 2 (hosted) hypervisors run on a conventional operating system just as other computer programs do. They abstract guest operating systems from the host operating system. For example, VMware Workstation and Oracle VirtualBox are Type 2 hypervisors. 


Containers are a form of operating system virtualization. A container contain the app and resources it needs to run. Containers are isolated from each other and the host system. They are lightweight and start quickly. Examples of containerization platforms include Docker and Kubernetes.




Virtual NIC: Software associated with unique MAC addresses which can be used by a VM to send and receive packets

Virtual Switch:  A Software that can connect to other virtual switchies and physical NICs to allow VMs to communicate with each other and the outside world





## VRF

VRF stands for Virtual Routing and Forwarding. It is a feature in networking that allows multiple routing tables to exist within a single router, enabling the separation of network traffic into distinct logical networks.

example:

conf t

vrf defubutuion TENTant a
address-family ipv4 
exit

vrf definition Tenant b
address-family ipv4
exit
int g0/1
vrf forwarding Tenant a
ip address 192.0.2.1 255.255.255.0
no shutdown
int g0/2
vrf forwarding Tenant a
ip address 10.1.1.1 255.255.255.0
no shutdown
int g0/3
vrf forwarding Tenant b
ip address 10.1.1.1 255.255.255.0
no shutdown
int g0/4
vrf forwarding Tenant b
ip address 198.51.100.1 255.255.255.0
no shutdown
exit
show ip route vrf Tenant a
conf t

int lo0
ip address 5.5.5.5 255.255.255.255

