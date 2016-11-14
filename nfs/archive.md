#Scenario - NFS ISO Archive
The customer XYZ is using an NFS archive for their software packages that are needed to generate ISO images. Since the workflows on their current legacy systems are very manual labor intensive they are looking at ECS to simplify their model and lower the cost per GB.

The workflow consists of an archive writer that commits software packages to the NFS share and the reader clients that pull specific packages and generate ISO images. Writing is always done by a single host with a pre-specified user (UID/GID = writer/writers).

Reading is done by multiple client systems from anywhere in the network. The application pulling the data is usually run as root/root, but credentials can vary.

It’s a mandatory requirement that only the writer can manipulate data on the share and that the reader(s), no matter their user credentials, can only read the data from the share.

To demonstrate the different aspects of the use case, the configuration will be done in steps to illustrate the different stages of the configuration.
The goal is ultimately to have a NFS export that can be written to by a writer user and be read(only) for all other users with varying credentials.

##Create the namespace
Begin with creating a local management user called *xyzadmin* with the password set to the same as the username. This user should not have any roles attached, which defaults it to a namespace admin once it’s aligned with a namespace.

Once the user is created, create a new namespace called *xyzarchive*, with the owner being *xyzadmin*.

Configure only the settings for the namespace that are necessary for the NFS testing. "Access During Outage" may be relevant if these tests are part of a larger suite of tests being performed.

##Create the object user
From the Manage section, go to users and select the *xyzarchive* namespace (if not autoselected) and create a new Object user with the name *writer*. As multiprotocol access is not required for this use-case, simply select *Save* once the name for the user has been entered. 
> NOTE: If *Next to add passwords* is clicked, the user will also automatically save and go to the credential screen for the user.

##Create the bucket
It's done under *manage->buckets->New Bucket*. It is possible to pre-select the xyzarchive namespace at this level, which will prepopulate it in the New Bucket page.

Name the bucket *xyzarchivebucket*, validate that the namespace is *xyzarchive* and ensure that the Bucket Owner is the previously created object user, *writer*. Below this select *Enable* for File System and fill in *writers* as the group name and allow read/write in both columns.

CBNOTE: leave the Default Bucket Group and permissions blank. If the setup is a GEO ECS, the ‘Access During Outage’ can be enabled if the Replication Group contains multiple sites. No other bucket settings should be set.

##Prepare the linux host
Before configuring the export on the ECS, user(s)/group(s) and mount point for the share is needed on the client.
Logging onto the linux host and switching to root, create the group *writers* with GID *5000*;
```
(sudo) groupadd -g 5000 writers
```

Next, create the user *writer* with the UID *5010* and with */home/writer* as the home directory:
```
(sudo) useradd -u 5005 -g 5000 -m -d /home/writer writer
```

As root, create the mountpoint */tmp/nfsmount*, which is where we will mount the export later:
```
sudo mkdir /tmp/nfsmount
```


##Mapping user and group
Before the newly created bucket can be exported, user/group mapping needs to be completed and an export defined.
Under *Manage->File*, select *User/Group Mapping* and create new mapping. 

First create a user mapping (slider bar in the bottom of the page) for the username *writer* and the UID *5005*. 

This will map the object user *writer* to a linux auth_sys UID of *5005* which corresponds to the linux user *writer* on the linux system.

Next, create a mapping for the group *writers* with a GID of *5000*. Remember to select *group* in the slider in the bottom part of the create mapping page.

##Creating the export
From the existing *User/Group Mapping* page, click the *Exports* button and then select *New Export*.

On the New File Export page, select *xyzarchive* as the namespace and then select the populated *xyzarchivebucket* from the dropdown below.

The export path below should state */xyzarchive/nfsarchivebucket/*. 

Click the *Add* button and bring up the export options popup.

The export host parameter signifies which hosts (comma separated IP addresses) can access the export. Put a wildcard (*) into this field to allow any host to access the export.

Select *Read/Write* under permissions. Let the transfer policy remain on Async unless there is a specific reason to use sync.

> NOTE, By the nature of NFSv3, async is more performant but less fault tolerant, see admin guide for the specifics in the section *Node and Site failure*. 

Select *SYS* as the authentication method, and leave the different flavors of Kerberos unselected. In this use case sub-directory mounting will not be required and thus the option will be at the default, not allowed. 

Leave the anonuser, anongroup and rootsquash empty as we will return to these later. Click *Add* and then *Save*, and the export will be complete.

##Validate that the export exists
On the linux host we can validate that the export has been created correctly by listing the exports on the ECS system using `showmount -e ECSIPADDR`

A list of all exports on the ECS will be listed. This is not only limited to a single namespace, so if there is a security requirement to prevent this listing, block the RPC call specifically in a firewall. The output will look like this:

```
/xyzarchive/xyzarchivebucket *
```

##Mount the export on the client
At this point we can mount the ECS export on the linux host as root, by running the following command:

```
(sudo) mount -t nfs -o "vers=3,nolock,sec=sys" ECSIP:/xyzarchive/xyzarchivebucket/ /tmp/nfsmount/
```

This will mount the share of type *nfs* and use the options (-o) *vers=3* for NFSv3, *nolock* to avoid locking issues and security *auth_sys* which defines which security model the share will use.

> While ECS does support locking via the NLM protocol, this is not required for this use case as it comes with specific architectural concerns. 

>Common options for mounts are also read/write size, ECS will as a default set it to 512 for both if the client supports it. For the sake of performance tuning, this can be changed using the r/wsize options as follows. 
>
Example: `vers=3,nolock,rsize=32768,wsize=32768,sec=sys`

##Validate the NFS mount
Validate that that you can now see that there is a NFS share mounted by running `ls -lrt /tmp/` which should now show the nfsmount as owned by the user *writer* and the group *2147483647* as seen here:
```
drwx---rwx 3 writer 2147483647    96 Sep 28 13:24 nfsmount
```

The reason why the group has a long integer instead of the related group name for *writer* which is *writers* is due to the *Default Bucket Group* has not been assigned at creation. If this had been done the output would look like this:
```
drwx---rwx 3 writer writers    96 Sep 28 13:24 nfsmount
```


Switch user from root to writer, enter in the *nfsmount* share and validate that it can be accessed and that the directory is empty:

```
root@linuxhost:/tmp# su - writer
writer@linuxhost:/tmp$ cd nfsmount/
writer@linuxhost:/tmp/nfsmount$ ls -lrt
total 0
```

##Write a file
Validate that a file can be written to the share, the directory listed to reflect the file created and that the contents of the file can be printed to screen:

```
writer@linuxhost:/tmp/nfsmount$ echo "The first file created by writer!" >> firstfile.txt
writer@linuxhost:/tmp/nfsmount$ ls -lrt
total 1
-rw-r--r-- 1 writer writers 33 Sep 28  2016 firstfile.txt
writer@linuxhost:/tmp/nfsmount$ cat firstfile.txt
The first file created by writer!
``` 


#Validate permissions
To validate that the permissions are indeed setup so that only the user writer can access the export, change back to the root user and access the share by changing directory to */tmp/nfsmount*.

From this you will receive an error *stale file handle* with a lot of surrounding question marks, as root does not have permission, this is expected behaviour of ECS NFS.

Logging back into writer and then changing to the same directory, validate that access is still there.

Next up is the configuration items required to ensure that *root* can read from *xyzarchive* and that any other user outside of *root* can do the same.

To do this we require a new object user, called *reader*. Follow the same steps as before to create it. As this object user will be mapped to anonymous users, it’s required that the object user has permissions on the bucket. 

From the bucket page, click the dropdown arrow next to edit and select *edit ACL*. From here, add the new object user under the User ACLs page, and assign it read/read acl only.

As we now have an object user which is defined with read access only, only the mapping remains. To map root users two steps needs to be completed, firstly the mapping of *reader* to UID 0, secondly adding the anon-id on the export.

Log out of the writer user (or su to root). Go to the mount, list the contents, read a file and then attempt to write a file as root:

```
root@linuxhost:/tmp# cd nfsmount/
root@linuxhost:/tmp/nfsmount# ls -lrt
total 1
-rw-r--r-- 1 writer writers 33 Sep 28 13:48 firstfile.txt
root@linuxhost:/tmp/nfsmount$ cat firstfile.txt
The first file created by writer!
root@linuxhost:/tmp/nfsmount# touch rootfile.txt
touch: cannot touch ‘rootfile.txt’: Permission denied
root@linuxhost:/tmp/nfsmount#
```

At this point we have write access for the user ‘writer’, we have read access for root - but what about other, random users?

Create the user *random* on the linux host by running:

```
(sudo) useradd -u 5007 -g 5007 -m -d /home/random random
```

This will create the user and group on the node.

Switch to the random user and rerun the above test:

```
random@linuxhost:/tmp/nfsmount# ls -lrt
total 1
-rw-r--r-- 1 writer writers 33 Sep 28 13:48 firstfile.txt
random@linuxhost:/tmp/nfsmount$ cat firstfile.txt
The first file created by writer!
random@linuxhost:/tmp/nfsmount# touch randomfile.txt
touch: cannot touch ‘randomfile.txt’: Permission denied
random@linuxhost:/tmp/nfsmount#
```

This concludes the archive scenario.
