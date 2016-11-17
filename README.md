# ECS 401 training labs

The ECS 401 training is focusing on hands-on labs to allow participants to become more familiar with different  ECS features and use cases. 

- [S3, Swift and Atmos APIs](apis/apis.md)
- [ECS-Sync](ecssync/ecssync.md)
- [Spark](spark/spark.md)
- NFS
	- [Archiving use case](nfs/archive.md)
	- [Multi protocol access use case](nfs/multiproto.md)
- [Load balancers](lb/lb.md)
- [COSBench](cosbench/cosbench.md)

This training is leveraging the ECS 3.0 VLAB.

The environment is composed of 2 ECS systems deployed as 3 different nodes (192.168.1.11-13 and 192.168.1.21-23). A program is available on the Desktop to check the DT status of both ECS VDCs.

All the Docker containers used in the training must be started in the Spark VM. You can connect to this VM (192.168.1.30) using *mRemoreNG*.

In the Spark VM, the Linux firewall is enabled by default. Before starting any training, you need to stop and disable it:
```
systemctl stop firewalld
systemctl disable firewalld
```

Before starting a new lab, you need to make sure that no other container is running.

Run the `docker ps` command to list the container that are currently running and `docker rm -f <container Id>` to delete a container.

If you get any *IPTABLES* related error when starting a new Docker container, you need to restart the Docker service by running the `systemctl restart docker` command.

This training has been developed by Cristina Alvarez, Christoffer Braendstrup and Denis Jannot.