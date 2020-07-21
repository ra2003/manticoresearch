# Restarting a cluster 

A multi-master replication cluster requires its single node to be started as a reference point before all the other nodes join it and form a cluster.  This is called cluster bootstrapping which introduces a `primary component` before others see that as a reference point to sync up the data from. The restart of a single node or reconnecting from a node after a shutdown can be done as usual.

After the whole cluster shutdown the server that was stopped last should be started first with `--new-cluster` command line option. To make sure  that the server is able to start as a reference point the `grastate.dat` file located at the cluster [path](Creating_a_cluster/Setting_up_replication/Setting_up_replication.md#Replication-cluster)  should be updated with the value of `1` for `safe_to_bootstrap` option. I.e., both conditions, `--new-cluster` and `safe_to_bootstrap=1`, must be satisfied. An attempt to start any other node without these options set will trigger an error. To override this protection and start cluster from another server forcibly, `--new-cluster-force` command line option may be used.

In case of a hard crash or an unclean shutdown of all the servers in the cluster you need to identify the most advanced node that has the largest `seqno` in the `grastate.dat` file located at the cluster [path](Creating_a_cluster/Setting_up_replication/Setting_up_replication.md#Replication-cluster) and start that server with the command line key `--new-cluster-force`.