# Brocade vTM & NGNIX Load Balancers with ECS#
---
DellEMC Elastic Cloud Storage (ECS) is a software-defined, cloud-scale, object storage platform that combines the cost advantages of commodity infrastructure with comprehensive protocol support for unstructured (Object and File) workloads.

In an optimal configuration, a load balancer is recommended to distribute the load across the nodes within ECS and ECS clusters in different locations, especially when using S3 or NFS.  

This document focuses on how to deploy and configure Brocade vTM and NGNIX load balancers. However, most of the ideas described in this document can be applied to other load balancer models.

The goal of this lab is to learn how to configure Brocade Virtual Traffic Manager (vTM)  and NGNIX Load Balancers to provide high availability for different access methods (S3, NFS, …) and topologies (single site or multi sites, …).

## Prerequisites

### ECS

If you don't have access to an ECS system, you can create an account on [ECS Test Drive](http://portal.ecstestdrive.com).

For this lab, please use the ECS F&F deployed in your lab:

```
- ecs-1-1.vlab.local: 192.168.1.11
- ecs-1-2.vlab.local: 192.168.1.12
- ecs-1-3.vlab.local: 192.168.1.13
```

### Load Balancers 

In this case, we'll use a Brocade vTM Docker container, already loaded in your Spark VM.
```
spark.vlab.local: 192.168.1.30
```

In general, this Docker containers may be downloaded from DockerHub:

```
docker pull djannot/brocade-vtm
docker pull djannot/haproxy
```

The components of a (virtual) load balancer farm are:
- One or more front-end machines running the load balancer software
- A number of back-end servers (ECS nodes in our case)

The front-end machines must be able to receive traffic from the Internet (or where the remote clients are located). They must also be able to contact the back-end machines.

The back-end servers will usually be visible only from an internal network. The front-end machines do not need to route traffic between the Internet and the back-end machines.

The load balancer software is commonly deployed on a multi-homed machine. One network interface card is visible to the Internet; one or more network interface cards are exposed to the internal private networks where the back-end servers reside. In this lab, we'll use a VM with with a single network card (this is common in an evaluation or testing environment), with connectivity to both the clients and the servers.


# Load Balancer's Lab


## Brocade vTM & ECS

In this lab, we'll deploy a single vTM appliance; although the best practice is to deploy at least 2 x vTM appliances per site in order to provide high availability and site-failover redundancy.

In the vTM world, there are two important config components:
- The VIP, where the vTM listen at. It is usually associated with a DNS record. 
    - We'll use *lb.vlab.local*, with the IP of your Spark VM *192.168.1.30*
- The vTM pool, with the back-end servers 
    - In this case, the available ECS nodes *192.168.1.11-13*

The vTM Virtual Server will listen to this VIP and balance the traffic across the available ECS nodes configured in the corresponding vTM Pool.


### S3 Load Balancing

In this first exercise, we'll configure the Brocade vTM, listening at *lb.vlab.local*, port 9020, redirecting requests to the ECS nodes (vTM pool), port 9020.

For  detailed information about how to create Virtual Servers or vPools, please, refer to the [ECS & Brocade Load Balancers Test Plan](https://inside.dell.com/docs/DOC-218468). It includes screenshots that could help with the configuration.

**Configuration tasks**

![vTM - S3 Load balancing](s3_1.jpg)

- Disable the firewall
> `systemctl stop firewalld`

- Start the Docker container
> `docker run -e ZEUS_EULA=accept -e ZEUS_PASS=password --privileged -t --net=host -d djannot/brocade-vtm`

- Open a web browser *http://192.168.1.30:9090*
    - Login with *admin/password* and click on *Use developer mode*

- Create a pool using the ECS node IPs
    - Use the right port for S3 (9020)
    - Use the right Health Monitor (Simple HTTP)
 
> ![vPool Creation](s3_0.jpg)	

- Create a Virtual Server, listening at port 9020, redirecting the traffic to the pool you just created. Make sure your Virtual Server is enabled.

- Create a DNS Host record for your virtual server -> lb.vlab.local 
	- The ECS already has a DNS record (ecs.vlab.local)
	- You can launch the Domain Controller from mRemoteNG; it's already preconfigured.

>**Note - S3 Ping: ** This exercise doesn't cover the Health monitoring. 
> It's a best practice to use a L7 ping, since it allows application level health status and not to simple detect if a node is alive.
> Brocade has developed a script to monitor the ECS nodes using L7 ping. For more information about it, please, check the [ECS & Brocade Load Balancers Test Plan](https://inside.dell.com/docs/DOC-218468), Section 6 -	Health check.

**Verification**

- Use S3 browser to test you can access your data using both, an ECS node and the Virtual server (IP or DNS record) you just created
	- Storage Type: S3 compatible storage
	- Create an user/bucket in ECS
	- Access key ID =  ECS object user name
	- Secret Access key =  ECS secret key
	- REST endpoint = ECS node/Load Balancer:9020
	- Verity that S3 Browser is configured for HTTP
		- Browser Tools -> Options -> Connection -> Uncheck *Use Secure Transfer (HTTPS)*
	
### SSL configuration

When secure communication is required, it is a best practice to offload and terminate SSL in the Load Balancer in order to not add extra load on ECS Nodes to handle SSL certificates.  Thus, the certificate generated needs to be installed on your load balancer.  If SSL termination is required on ECS nodes itself (end-to-end SSL encryption), then use load balancing mode to pass through the SSL traffic to ECS nodes for handling.  In this scenario, the certificates would need to be installed on ECS.

A second best practice is to use a certificate signed by a Certificate Authority (CA). Self-signed certificate can also be used, but are not recommended for production environments. The Common Name (CN) of the certificate should match the endpoint (lb.vlab.local in our case). Usually, the CN is using a wildcard at the beginning (*.lb.vlab.local ) in order to be compatible with Virtual Host style S3 adressing (ex: bucket1.lb.vlab.local ).

In this exercise, we'll go through the first scenario, SSL offloading at the Load Balancer using a self-signed certificate.


**Configuration tasks**

In this second exercise, we'll configure the Brocade vTM, listening at *lb.vlab.local*, port 9021 (secured S3), redirecting requests to the ECS nodes (vTM pool), port 9020.

![vTM - S3 SSL Load balancing](s3_2.jpg)

- Create a Wildcard in your DNS for your load balancer:
	- Create a wildcard **.lb.vlab.local* and verify that you can ping *bucket.lb.vlab.local*
	- You can launch the Domain Controller from mRemoteNG; it's already preconfigured.

>> ![DNS wildcard](wildcard.png)

- Modify your Virtual Server configuration to use SSL. 
	- Listening at port 9021 (HTTP) 
	- Go to SSL Decryption
        	- Enable SSL Decryption
		- Create a certificate:
        		- Manage SSL certificates -> Create Self-Signed Certificate -> Certificate Signing Request. The Name should be lb.vlab.local and the Common Name should match with the wildcard (*.lb.vlab.local)

**Verification**

- Use S3 browser to test you can access your data using port 9021
	- Modify the REST endpoint configuration in your S3 Browser account = lb.vlab.local:9021
	- Verity that S3 Browser is configured for HTTPS
		- Browser Tools -> Options -> Connection -> Check *Use Secure Transfer (HTTPS)*


### NFS Load Balancing

ECS supports NFS v3 which is generally configured in asynchronous mode.
The NFS clients sends many data packets and regular commits for this data.
For that reason, the NFS clients must send the data packets and the commits to the same ECS node.
In order to provide high availability, a load balancer should be used, but configured with session persistence.

In addition, we need to take into consideration that NFS leverages different daemons (nfsd, mountd, nlockmgr and portmapper) and ports (111, 2049 and 10000 UDP/TCP).

**Configuration tasks**

In this third exercise, we'll configure the Brocade vTM, listening at *lb.vlab.local*, NFS ports, redirecting the traffic to the ECS nodes (vTM pool), NFS ports, using sticky sessions.

![vTM - NFS Load balancing](nfs_1.jpg)

- Create Virtual Pools and Virtual Servers for the TCP and UDP ports used in NFS (111, 2049 and 10000). 
	- Configure TCP Virtual Servers as *Generic Streaming* 
	- Configure UDP Virtual Servers as *UDP - Streaming*
	- Use *ping* as Health Monitor for your vPools

> At the end you should get something like this:

> ![vTM - NFS Load balancing](nfs_2.jpg)

- Configure sticky sessions in your NFS pools
	- Create a session persistence configuration at the IP level and associate it with your NFS pools.

**Verification**

- Use your Linux client and mount your ECS export using the Virtual Server you just created.

## NGINX

NGNIX is an open source; reliable and free load balancing software solution. It provides a low-cost option for customers who desire to utilize a load balancer with ECS. 

### S3 Load Balancing

**Configuration tasks**

In the NGNIX section, we'll reuse the same DNS record for the Load Balancer VIP, using different ports to avoid conflicts in your Spark VM. In a real environment, any port would work.

![NGINX - S3 Load balancing](ngnix_1.jpg)

- Disable selinux by executing the `su -c "setenforce 0"` command to avoid a permission issue in the Docker container

- Create a `/root/nginx` directory

- Create the `/root/nginx/nginx.conf` file with the following content:
```
worker_processes 4;

events {
}

http {

  upstream ecs {
    server 192.168.1.11:9020;
    server 192.168.1.12:9020;
    server 192.168.1.13:9020;
  }

  server {
    listen 80;

    location / {
      sendfile on;
      tcp_nopush on;
      tcp_nodelay on;
      client_max_body_size 1024G;
      proxy_buffering off;
      proxy_buffer_size 4k;
      proxy_pass http://ecs;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
  }
}

```  
- Start the Docker container

`docker run --net=host --name nginxlb -v /root/nginx:/etc/nginx -d nginx`

**Verification**

- Use S3 browser to test that you can access your data using port 80
	- Modify the REST endpoint configuration in your S3 Browser account = lb.vlab.local:80
	- Verity that S3 Browser is configured for HTTP
		- Browser Tools -> Options -> Connection -> Uncheck *Use Secure Transfer (HTTPS)*

### SSL configuration

**Configuration tasks**

Finally, let's configure NGNIX to offload the SSL complexity in a secure scenario, using again a self-signed certificate.

![NGNIX - S3 SSL Load balancing](ngnix_2.jpg)

- Create a `/root/nginx/ssl` directory

- Create a certificate for the Common Name **.lb.vlab.local*
    - You can either create a new certificate using OpenSSL
    	- /etc/nginx/ssl/server.crt
    	- /etc/nginx/ssl/server.key
    - Or reuse the certificate you created in the previous section with Brocade, since you are using the same CN = **.lb.vlab.local*
    	-  In order to get the certificate created in the vTM, run:
```
docker ps #Get the containier ID
docker cp <container id>:/usr/local/zeus/zxtm-11.0/conf_A/ssl/server_keys/lb.vlab.local.public /root/nginx/ssl/server.crt
docker cp <container id>:/usr/local/zeus/zxtm-11.0/conf_A/ssl/server_keys/lb.vlab.local.private /root/nginx/ssl/server.key
```
- Add the following lines to the `server` section of the `nginx.conf` file:

```
listen 443 default_server ssl;
ssl_certificate         /etc/nginx/ssl/server.crt;
ssl_certificate_key     /etc/nginx/ssl/server.key;
```

- Restart the Docker container

> `docker restart nginxlb`

**Verification**

- Use S3 browser to test that you can access your data using port 443
	- Modify the REST endpoint configuration in your S3 Browser account = lb.vlab.local:443
	- Verity that S3 Browser is configured for HTTPS
		- Browser Tools -> Options -> Connection -> Check *Use Secure Transfer (HTTPS)*

## References

- [ECS & Brocade Load Balancers Test Plan](https://inside.dell.com/docs/DOC-218468)
- Brocade vTM - http://www.brocade.com/en/products-services/software-networking/application-delivery-controllers/virtual-traffic-manager.html
- NGNIX Wiki - https://www.nginx.com/resources/wiki/