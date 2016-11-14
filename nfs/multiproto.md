#Scenario - Multi-protocol access
XYZ has a department that stores large quantities of sensor data from product development tests. The tests are uploaded to a USB disk onsite and sent into a local branch office where it’s transferred to a centrally located CIFS share. While it has not been a priority earlier, just retaining the sensor data for logging purposes, they are now interested in future analytics of the data. Having been told of the ECS multi-protocol access capabilities they are looking to consolidate this use case onto ECS, allowing for data to be uploaded directly from a CIFS client (via S3) to ECS or via using a tool like S3Browser to upload it and then ensure that the data is accessible internally via NFS.

##Requirements
For this scenario a windows host loaded with S3 Browser and a linux host for mounting the NFS share onto is needed.
While there are no requirements for any specific users on the windows host there are on the linux host. Utilizing S3 which carries its own set of credentials for connectivity no specific authentication is required apart from a relevant linux user/group mapped on the ECS. 

> NOTE - As S3 is not NFS permission aware, ECS works around this by adding a Default Bucket Group (DBG) to a bucket and also aligns the bucket owner/object user with local mapping against a linux user id. This means that when ECS writes to the bucket from S3, the DBG will be applied to the data and additionally since there is a mapping between the object user and the linux UID, the permissions will match the NFS linux user along with the group. 

##Configure
Having completed the NFS Archive lab the steps for this test will be based on the experience gained and will not be detailed to the same extent.

###Prepare a namespace
Create a management user called *xyzmpasadmin* and a namespace called *xyzmpas* (Multi Protocol Access Storage) owned by this management user.

###Prepare the object user
Create an object user with S3 credentials called *mpasuser*. 

Once created, a new bucket *mpasbucket* is next. Set the owner to *mpasuser* with File enabled and the DBG configured to *mpasgroup* with read/write permissions. 

###Perform the mapping and export
When the user and bucket have been configured, create user and group mappings *mpasuser -> UID 6010* and *mpasgroup -> GID 6000*.

Next, configure the NFS export for the bucket with a wildcard for hosts (*), read/write, async and save. Validate that the export is seen on the host using showmount.

###Prepare the client and test access
On the linux host, create a mount directory, */tmp/mpasmount* and create a user and group that matches the above. 

Ensure that the UID/GID matches the mapping configured on ECS and mount the export as root, switch user to *mpasuser* and validate that the export can be accessed by reading the directory and uploading a text file.

Commands needed:
```
(sudo) groupadd -g 6000 mpasgroup
(sudo) useradd -u 6010 -g 6000 -m -d /home/mpasuser mpasuser
(sudo) mount -t nfs -o "vers=3,nolock,sec=sys" ECSIP:/xyzmpas/mpasbucket/ /tmp/mpasmount/
```

Once the text file has been created, use a S3 client (s3browser, cloudberry, s3curl, etc.) to validate the object users connectivity. Once connected, validate that the file uploaded via NFS is seen, download it, validate the content of the file and apply new lines of text within and reupload it without renaming it. 
When the upload is complete, cat the file from the NFS export and validate the update. 

This concludes the multi-protocol access scenario.