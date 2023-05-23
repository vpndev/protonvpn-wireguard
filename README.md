<!-- TOC -->

- [ProtonVPN - Wireguard readme](#protonvpn---wireguard-readme)
  - [NAT-PMP Port Forwarding](#nat-pmp-port-forwarding)
    - [Issues to be fixed in ProtonVPN's guide for manual Port Forwarding setup](#issues-to-be-fixed-in-protonvpns-guide-for-manual-port-forwarding-setup)
    - [Solution to mentioned issues](#solution-to-mentioned-issues)
      - [Check if port forwarding is enabled](#check-if-port-forwarding-is-enabled)
      - [Create a UDP port mapping on UDP, needed for port forwarding](#create-a-udp-port-mapping-on-udp-needed-for-port-forwarding)
      - [loop natmpmpc so that it doesn’t expire](#loop-natmpmpc-so-that-it-doesnt-expire)
        - [loop result](#loop-result)

<!-- /TOC -->

# ProtonVPN - Wireguard readme

ProtonVPN - Wireguard

ProtonVPN is considered as very popular VPN provider and since wireguard protocol became available, I decided to test paid servers and check its abilities and features.

## NAT-PMP Port Forwarding

One of main disadvantages of protonvpn compared to some other vpn service is port forwarding. Proton released a feature where one port is opened dynamically and changes every 60 seconds. Its use with original ProtonVPN client is made in some way usable. If wireguard's native client is used, then I found [following guide in proton vpn documentation](https://protonvpn.com/support/port-forwarding-manual-setup/). Following that guide at the time of creation of this document did not work and here I will share some unmentioned information which is at the time of creation of current document not part of that guide.

Main problem in guide is that professional will be able to quickly find out why commands do not/can not work, but for normal user this might be unresolable problem especially since there are few parts in that guide which require corrections:

### Issues to be fixed in ProtonVPN's guide for manual Port Forwarding setup 

1. Servers where port forwarding is available? / **double-arrow icon**
   in ***[Step 1: Configure VPN Settings](https://protonvpn.com/support/port-forwarding-manual-setup/)*** it says:
   
   *All our P2P servers support port forwarding. P2P servers can be easily identified in our apps and on our VPN configuration download pages by a double-arrow icon.*

   - First proton claims that **all P2P servers support port forwarding** but **does not mention if servers without double-arrow icon work** if switch "NAT-PMP" is enabled?
   - Then proton says how P2P servers can be easily identified by double-arrow icon indicating that only servers with double-arrow icon work

   Both statements are misleading and wrong.

1. **If this test fails (see screenshot below), please return to Step 1 of this guide,**

   If test fails, then returning to Step 1, then it will not help much if commands used are wrong. In all my test "using gateway: ..." results in default gateway of network interface. Here one example, where network interface is in 192.168.10.0/24 subnet and gateway's ip is 192.168.10.1:

    ```bash
    root@demo:~# natpmpc
    initnatpmp() returned 0 (SUCCESS)
    using gateway : 192.168.10.1
    sendpublicaddressrequest returned 2 (SUCCESS)
    readnatpmpresponseorretry returned -7 (FAILED)
    readnatpmpresponseorretry() failed : the gateway does not support nat-pmp
        errno=111 'Connection refused'
    ```

    Following ProtonVPN's guide in returning to step 1 would bring nothing as it does not matter which server was selected, with or without double-arrow icon as the problem is that default gateway will in most cases be not the gateway ip from proton's server peer.

    Sending a user to step 1 is even more confusing me, especially since I selected a server "without double-arrow icon", it is only important that NAT-PMP is enabled during config generation.
    
    If one wants to check how natpmpc is used, run `natpmpc --help` and you can see how to set correct gateway:
    
    ```text
    Option available :
      -g ipv4address
	force the gateway to be used as destination for NAT-PMP commands.
    ```

1. UDP port mapping
   
   Command `natpmpc -a 0 0 udp 60` will fail due to same reason as previously explained => wrong gateway.

   ```bash
   root@demo:~# natpmpc -a 0 0 tcp 60
   initnatpmp() returned 0 (SUCCESS)
   using gateway : 192.168.10.1
   sendpublicaddressrequest returned 2 (SUCCESS)
   readnatpmpresponseorretry returned -7 (FAILED)
   readnatpmpresponseorretry() failed : the gateway does not support nat-pmp
       errno=111 'Connection refused'
   ```

1. Wrong address 10.16.0.1
   
   In guide `10.16.0.1` is used as gateway, but all wireguard configuration have `10.2.0.2/32` assigned with gateway ip: `10.2.0.1`:

   ```bash
   root@demo:~# natpmpc -g 10.16.0.1
   initnatpmp() returned 0 (SUCCESS)
   using gateway : 10.16.0.1
   sendpublicaddressrequest returned 2 (SUCCESS)
   readnatpmpresponseorretry returned -100 (TRY AGAIN)
   readnatpmpresponseorretry returned -100 (TRY AGAIN)
   readnatpmpresponseorretry returned -100 (TRY AGAIN)
   ```

### Solution to mentioned issues

Use option `-g` to set default gateway whe using `natpmpc`.

Example with Austrian *secure core server (CH-AT)*:

If one sets correctly a gateway to correct address which is 10.2.0.1, then the result is successfull, first a test:

#### Check if port forwarding is enabled

```bash
root@demo:~# natpmpc -g 10.2.0.1
initnatpmp() returned 0 (SUCCESS)
using gateway : 10.2.0.1
sendpublicaddressrequest returned 2 (SUCCESS)
readnatpmpresponseorretry returned 0 (OK)
Public IP address : 87.249.133.98
epoch = 108573
closenatpmp() returned 0 (SUCCESS)
```

#### Create a UDP port mapping on UDP, needed for port forwarding

```bash
root@demo:~# natpmpc -g 10.2.0.1 -a 0 0 udp 60
initnatpmp() returned 0 (SUCCESS)
using gateway : 10.2.0.1
sendpublicaddressrequest returned 2 (SUCCESS)
readnatpmpresponseorretry returned 0 (OK)
Public IP address : 87.249.133.98
epoch = 109279
sendnewportmappingrequest returned 12 (SUCCESS)
readnatpmpresponseorretry returned 0 (OK)
Mapped public port 59800 protocol UDP to local port 59800 liftime 60
epoch = 109279
closenatpmp() returned 0 (SUCCESS)
```

#### loop natmpmpc so that it doesn’t expire

```bash
while true ; do date ; natpmpc -g 10.2.0.1 -a 0 0 udp 60 && natpmpc -g 10.2.0.1 -a 0 0 tcp 60 || { echo -e "ERROR with natpmpc command \a" ; break ; } ; sleep 45 ; done
```

##### loop result
```bash
root@demo:~# while true ; do date ; natpmpc -g 10.2.0.1 -a 0 0 udp 60 && natpmpc -g 10.2.0.1 -a 0 0 tcp 60 || { echo -e "ERROR with natpmpc command \a" ; break ; } ; sleep 45 ; done
Di 23. Mai 03:30:50 CEST 2023
initnatpmp() returned 0 (SUCCESS)
using gateway : 10.2.0.1
sendpublicaddressrequest returned 2 (SUCCESS)
readnatpmpresponseorretry returned 0 (OK)
Public IP address : 87.249.133.98
epoch = 115058
sendnewportmappingrequest returned 12 (SUCCESS)
readnatpmpresponseorretry returned 0 (OK)
Mapped public port 51637 protocol UDP to local port 51637 liftime 60
epoch = 115058
closenatpmp() returned 0 (SUCCESS)
initnatpmp() returned 0 (SUCCESS)
using gateway : 10.2.0.1
sendpublicaddressrequest returned 2 (SUCCESS)
readnatpmpresponseorretry returned 0 (OK)
Public IP address : 87.249.133.98
epoch = 115058
sendnewportmappingrequest returned 12 (SUCCESS)
readnatpmpresponseorretry returned 0 (OK)
Mapped public port 51637 protocol TCP to local port 51637 liftime 60
epoch = 115058
closenatpmp() returned 0 (SUCCESS)
Di 23. Mai 03:31:35 CEST 2023
initnatpmp() returned 0 (SUCCESS)
using gateway : 10.2.0.1
sendpublicaddressrequest returned 2 (SUCCESS)
readnatpmpresponseorretry returned 0 (OK)
Public IP address : 87.249.133.98
epoch = 115103
sendnewportmappingrequest returned 12 (SUCCESS)
readnatpmpresponseorretry returned 0 (OK)
Mapped public port 51637 protocol UDP to local port 51637 liftime 60
epoch = 115103
closenatpmp() returned 0 (SUCCESS)
initnatpmp() returned 0 (SUCCESS)
using gateway : 10.2.0.1
sendpublicaddressrequest returned 2 (SUCCESS)
readnatpmpresponseorretry returned 0 (OK)
Public IP address : 87.249.133.98
epoch = 115103
sendnewportmappingrequest returned 12 (SUCCESS)
readnatpmpresponseorretry returned 0 (OK)
Mapped public port 51637 protocol TCP to local port 51637 liftime 60
epoch = 115103
closenatpmp() returned 0 (SUCCESS)
```