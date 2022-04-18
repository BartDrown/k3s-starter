
# k3s-starter
### This repository contains example setup for k3s cluster on vm's from scratch
#### Along with traefik ingress example and certificate issuer  

If you want more extensive guide, be sure to check out [this guide](https://github.com/hobby-kube/guide).
This is great resource, though it's not about k3s, and covers up nginx ingress instead of traefik ingress, so I decided to make my own helper.

Also be sure to check out [this post](https://sysadmins.co.za/https-using-letsencrypt-and-traefik-with-k3s/). Ruan made good work describing traefik ingress usage, without him I probably would be stuck with some wacky, not flexible configuration.

## Requirements:

 - At least one machine with public ip (it will serve as control-plane and master node).
 - Any number of machines that are accessible from machine with public ip.
 - Private internal network
 - Registered domain name

> You can of course setup k3s cluster just with one node, but it's not
> much of a prodocution solution.

> Private network is also not required if all of your machines have public ip addresses, but in that case, you would need to secure connection with internal vpn. More extensive guide about it can be find [here](https://github.com/hobby-kube/guide#wireguard-setup)

#### Chosing cloud providers
You can choose pretty much any cloud provider, just buy almost bare machines, do not buy ready kubernetes clusters cause they are expensive and they lack flexibility of configuration.

I've chosen 3 machines on Hetzner cloud with 2 GB RAM with Ubuntu. 
It's pretty cheap, got private internal network out of the box (just set it up in web panel) and got data centers pretty much close to where I live.

Here is [referral link](https://hetzner.cloud/?ref=osR7dA9R4bmz) for 20$ in their cloud.
(Not mine though, its for guy from guides I've attached, you can thank him that way)

> You can also try to set up cluster on {minikube](https://minikube.sigs.k8s.io/docs/), it's local cluster, good place to start you journey with kubernetes. Just be aware that it's not the way that it should work and you probably won't be able to get ssl certificates without something like ngrok and for sure, not for long.

#### Checking connection

Before starting, make sure that every machine can reach eachother, or at least every potential worker node machine, can reach machine which will become master node. 

Preferably you would use private network to do that, but if you don't have one and you made vpn connection as described above, you need to use addresses defined there. 

If you don't have private, neither vpn connection to other machines, you could try to do it over public interfaces, though it is not secure in my opinion and I would not recommend that.

To check connection you can use ping command:

    ping <other machine address> 

 

## Pre Configuration

Before we install k3s, we need to prepare our machines to be able to securely handle requests, but also secure remote login in good way. 

##### These are the steps that you need to perform on every node

You probably could skip firewall configuration on worker nodes, if they not exposed on public network, but it does not hurt and you would not need to remember about it further, when you would need for example multiple control planes for [HA cluster](https://medium.com/velotio-perspectives/demystifying-high-availability-in-kubernetes-using-kubeadm-3d83ed8c458b#:~:text=Kubernetes%20High-Availability%20is%20about,access%20to%20same%20worker%20nodes.).

#### Create secure user

At the start you should create non sudo user with ssh access and disable default root login. 

    adduser <username>
After that, you need to add this user to sudo group:

    usermod -aG sudo <username>

Change */etc/ssh/sshd_config* property to the following:

    PasswordAuthentication  no
    
Restart ssh service and logout:

    sudo systemctl restart ssh

**Make sure** that you can login with ssh, without password prompt, to new user account. 
If not, you probably misconfigured something and need to fall back to previous steps.
 
If you are absolutely sure that you can login with ssh to the new user, you can now disable default root login

Change */etc/ssh/sshd_config* property to the following:

    PermitRootLogin  no
    
Restart ssh service:

    sudo systemctl restart ssh
   
   Now you should be able to login into your new user account, with ssh only.

#### Prepare firewall

Firewall is must, we don't want to allow exposing services on some wacky ports, at least not out of the box.
Best solution for that is configuring [UncomplicatedFirewall](https://wiki.ubuntu.com/UncomplicatedFirewall).

Most important line is first, you need to allow ssh or in other case **you will lock your machine**.

    sudo ufw allow ssh
    suddo ufw allow 6443
    sudo ufw allow 6443
    sudo allow 80
    sudo uwf allow 80
    sudo ufw allow 80
    sudo ufw allow 443
    sudo ufw default deny incoming
    sudo ufw enable 

Also as you can see, you also need to allow ports:

 -  6443  - kubernetes api (allows you to connect with kubectl remotely and allows other remote nodes to talk with your master node)
 - 80 - http (I tend to not use http, but I redirect it to https usually with middleware)
 - 443 - https 

> You can allow pretty much any port if you wish, it shouldn't affect you that much, just remember more ports - more point of possible unwanted entry/misconfiguration. It's easier to manage firewall with less entrypoints. 

## Installing k3s

Now we can proceed with installing k3s on our master not-yet-node.
Master node it's simply node which holds control plane and wraps up kubernetes api for other nodes.

Just choose any machine with public ip that you wish to become your master node. 

Don't forget to point your dns record to it, for example like this:

| Host name | Type | TTL | Data |
|--|--|--|--|
| example.com | A | 1 hour | \<master-node ip-address\> |
| *.example.com | CNAME | 1 hour | example.com |

#### Starting up k3s (master node)

Substitute master node ip address in the following command and execute it:

    curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--tls-san <master-node ip-address>" sh -

Please note usage of "*-tls-san* \<master-node ip-address\>" parameter in execution variable. 
Its purpose it's to indicate that also our ip address is correlated with certificates that will be issued in the future, making them more secure and possibly not stealable/forgeable (at least I hope so).

After some time your master node should be up and you should be able to check if it's working with kubectl: 

    sudo kubectl get nodes
    
Command should list one node, with control-plane and master roles.

#### Joining k3s (working nodes)

This step is of course optional if you just want to have cluster with single node, but if you have more machines that you would like to use as nodes, you need to perform it.

We'll be now joining nodes to your cluster
First, you need to extract token from your master node:

    sudo cat /var/lib/rancher/k3s/server/node-token

Then proceed to machine which will become worker node and execute following command (don't forget to replace variables):

    curl -sfL https://get.k3s.io | K3S_URL=https://<master-node private ip-address>:6443 K3S_TOKEN=<extracted-token>

After short while, machine should join your cluster.

Go now to your master node and execute command: 

    sudo kubectl label node <worker-node-name> node-role.kubernetes.io/worker=worker

It should add worker role to your worker node, which would make it schedulable (in short word, it would allow master node to schedule pods into it in the future).

Now you can proceed to add other machines (if you have any) to your cluster in similar way.


You can check cluster status on your master node executing command:

    sudo kubectl get nodes

After you join your nodes into cluster, command output should be similar to this: 
![image](https://user-images.githubusercontent.com/40639741/163825254-2b1a6650-8408-4a09-849a-d6de9749a215.png)
