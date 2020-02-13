[参考文章](https://www.systutorials.com/docs/linux/man/8-heketi-cli/)

## SYNOPSIS

**heketi-cli** [commands] [options]  

## Cluster管理

```bash
heketi-cli cluster create Create a cluster 
    Example 
    $ heketi-cli cluster create
heketi-cli cluster delete <CLUSTER-ID> Delete a cluster 
    Example 
    $ heketi-cli cluster delete 886a86a868711bef83001
heketi-cli cluster info <CLUSTER-ID> Retrieves information about cluster

    Example 
    $ heketi-cli cluster info 886a86a868711bef83001

heketi-cli cluster list Lists the clusters managed by Heketi

    Example 
    $ heketi-cli cluster list
```
## Node管理

```bash
Node Commands
heketi-cli node add --zone=<ZONE-NUMBER> --cluster=<CLUSTER-ID> --management-host-name=<MANAGEMENT-HOSTNAME> --storage-host-name=<STORAGE-HOSTNAME> Add new node to be managed by Heketi
Options
--cluster=""  The cluster in which the node should reside

--management-host-name="" Management host name

--storage-host-name=""  Storage host name

--zone=-1 he zone in which the node should reside

Example 
    $ heketi-cli node add \ 
               --zone=3 \ 
               --cluster=3e098cb4407d7109806bb196d9e8f095 \ 
               --management-host-name=node1-manage.gluster.lab.com \ 
               --storage-host-name=node1-storage.gluster.lab.com

heketi-cli node delete <NODE-ID> Deletes a node from Heketi management

    Example (删除node需要先删除device)
    $ heketi-cli node delete 886a86a868711bef83001

heketi-cli node disable <NODE-ID> Disallow usage of a node by placing it offline

    Example 
    $ heketi-cli node disable 886a86a868711bef83001

heketi-cli node enable <NODE-ID> Allows node to go online

    Example 
    $ heketi-cli node enable 886a86a868711bef83001

heketi-cli node info <NODE-ID> Retrieves information about node

    Example 
    $ heketi-cli node info 886a86a868711bef83001

heketi-cli node list List all nodes in cluster

    Example 
    $ heketi-cli node list
```

## device 磁盘区管理

```bash
heketi-cli --user admin --secret jrtz0755 topology info
# 增加device
heketi-cli --user admin --secret jrtz0755 device add --name=/dev/sdc --node=33bbf6ab312fc54030d55da140992a97
# 删除device
heketi-cli --user admin --secret jrtz0755 device delete 313ebb2068f8efb240e82d331ec5748a
# 离线device: Disallow usage of a device by placing it offline
heketi-cli --user admin --secret jrtz0755 device disable 313ebb2068f8efb240e82d331ec5748a
# 上线device: Allows device to go online
heketi-cli --user admin --secret jrtz0755 device enable 313ebb2068f8efb240e82d331ec5748a
#  Retrieves information about device
heketi-cli --user admin --secret jrtz0755 device info 313ebb2068f8efb240e82d331ec5748a
```

## Volume 管理

```bash
heketi-cli volume create --cluster=<CLUSTER-ID> --disperse-data=<DISPERSION-VALUE> --durability=<TYPE> --name=<VOLUME-NAME> --redundancy=<REDUNDENCY-VALUE> --replica=<REPLICA-VALUE> --size=<VOLUME-SIZE> --snapshot-factor=<SNAPSHOT-FACTOR-VALUE> Create a GlusterFS volume
Options
--clusters=""
           Optional: Comma separated list of cluster ids where this volume
           must be allocated. If omitted, Heketi will allocate the volume
           on any of the configured clusters which have the available space.
           Providing a set of clusters will ensure Heketi allocates storage
           for this volume only in the clusters specified.
--disperse-data=4
           Optional: Dispersion value for durability type 'disperse'.
           Default is 4
--durability="replicate"
           Optional: Durability type.
                     Values are:
                     none: No durability. Distributed volume only.
                     replicate: (Default) Distributed-Replica volume.
                     disperse: Distributed-Erasure Coded volume.
--gid=0    Optional: Initialize volume with the specified group id
            Default is 0
--name=""  Optional: Name of volume. Only set if really necessary
--persistent-volume[=false]
         Optional: Output to standard out a persistent volume JSON file for OpenShift or
            Kubernetes with the name provided.
--persistent-volume-endpoint=""
           Optional: Endpoint name for the persistent volume
--persistent-volume-file=""
            Optional: Create a persistent volume JSON file for OpenShift or
            Kubernetes with the name provided.
--redundancy=2
            Optional: Redundancy value for durability type 'disperse'.
            Default is 2
--replica=3
            Replica value for durability type 'replicate'.
            Default is 3
--size=-1   Size of volume in GB
--snapshot-factor=1
            Optional: Amount of storage to allocate for snapshot support.
            Must be greater 1.0.  For example if a 10TiB volume requires 5TiB of
            snapshot storage, then snapshot-factor would be set to 1.5.  If the
            value is set to 1, then snapshots will not be enabled for this volume

Example 
* Create a 100GB replica 3 volume: 
$ heketi-cli volume create --size=100 
* Create a 100GB replica 3 volume specifying two specific clusters:
$ heketi-cli volume create --size=100 \ 
--clusters=0995098e1284ddccb46c7752d142c832,60d46d518074b13a04ce1022c8c7193c 
* Create a 100GB replica 2 volume with 50GB of snapshot storage:
$ heketi-cli volume create --size=100 --snapshot-factor=1.5 --replica=2 
* Create a 100GB distributed volume 
$ heketi-cli volume create --size=100 --durability=none 
* Create a 100GB erasure coded 4+2 volume with 25GB snapshot storage:
$ heketi-cli volume create --size=100 --durability=disperse --snapshot-factor=1.25 
* Create a 100GB erasure coded 8+3 volume with 25GB snapshot storage:
$ heketi-cli volume create --size=100 --durability=disperse --snapshot-factor=1.25 \ 
                 --disperse-data=8 --redundancy=3

heketi-cli volume delete <VOLUME-ID> Deletes the volume

Example 
$ heketi-cli volume delete 886a86a868711bef83001
heketi-cli volume expand --expand-size=<SIZE> --volume=<VOLUME-ID> Expand a volume
Options
--expand=""  Amount in GB to add to the volume

--volume="" Id of volume to expand

Example 
* Add 10GB to a volume ##扩容卷
$ heketi-cli volume expand --volume=60d46d518074b13a04ce1022c8c7193c --expand-size=10

heketi-cli volume info <VOLUME-ID> Retrieves information about volume

Example 
$ heketi-cli volume info 886a86a868711bef83001

heketi-cli volume list Lists the volumes managed by Heketi

Example 
$ heketi-cli volume list
```

## Topology命令

```bash
heketi-cli topology load --json=<JSON-FILENAME> Add devices to Heketi from a configuration file
Options
-j, --json="" Configuration containing devices, nodes, and clusters, in JSON format
  Example 
$ heketi-cli topology load --json=topo.json

heketi-cli topology info Retreives information about the current Topology

  Example 
$ heketi-cli topology info
```

### Setup OpenShift/Kubernetes persistent storage for Heketi

```bash
heketi-cli setup-openshift-heketi-storage Creates a dedicated GlusterFS volume for Heketi. Once the volume is created, a Kubernetes/OpenShift list object is created to configure the volume.
Options
--listfile="heketi-storage.json" Filename to contain list of objects

Example 
$ heketi-cli setup-openshift-heketi-storage
```

## GLOBAL OPTIONS

```bash
--json[=false] Print response as JSON

--secret="" 
Secret key for specified user.  Can also be set using the environment variable HEKETI_CLI_KEY

-s, --server=""
Heketi server. Can also be set using the
environment variable HEKETI_CLI_SERVER

--user=""
Heketi user.  Can also be set using the environment variable HEKETI_CLI_USER

-v, --version[=false] Print version
```

## EXAMPLE

```bash
$ export HEKETI_CLI_SERVER=http://localhost:8080
$ heketi-cli volume list
```

