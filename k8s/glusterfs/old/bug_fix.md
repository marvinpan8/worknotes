# BUG修复

## Afr: Avoid logging of "attempting to connect"

### 问题描述

/var/log/glusterfs/glustershd.log  输出

```bash
[2020-02-16 09:22:08.512457] E [socket.c:2524:socket_connect_finish] 0-vol_d049e83d6de65d955b3047bbeba3de40-client-1: connection to 192.168.10.113:49161 failed (Connection refused); disconnecting socket
[2020-02-16 09:22:08.516354] W [rpc-clnt.c:1753:rpc_clnt_submit] 4-vol_0f98a528def6f56252504c295581b4f0-client-2: error returned while attempting to connect to host:(null), port:0
[2020-02-16 09:22:08.516765] E [socket.c:2524:socket_connect_finish] 2-vol_50cfefdda6788f3c240dbc503591c71c-client-0: connection to 192.168.10.113:49159 failed (Connection refused); disconnecting socket
[2020-02-16 09:22:08.518143] W [rpc-clnt.c:1753:rpc_clnt_submit] 4-vol_0f98a528def6f56252504c295581b4f0-client-2: error returned while attempting to connect to host:(null), port:0
[2020-02-16 09:22:08.519839] I [rpc-clnt.c:2105:rpc_clnt_reconfig] 4-vol_0f98a528def6f56252504c295581b4f0-client-2: changing port to 49154 (from 0)
[2020-02-16 09:22:08.524058] E [socket.c:2524:socket_connect_finish] 4-vol_0f98a528def6f56252504c295581b4f0-client-2: connection to 192.168.10.122:49154 failed (Connection refused); disconnecting socket
[2020-02-16 09:22:08.677341] W [rpc-clnt.c:1753:rpc_clnt_submit] 0-vol_50cfefdda6788f3c240dbc503591c71c-client-1: error returned while attempting to connect to host:(null), port:0
[2020-02-16 09:22:08.893413] W [rpc-clnt.c:1753:rpc_clnt_submit] 0-vol_d049e83d6de65d955b3047bbeba3de40-client-2: error returned while attempting to connect to host:(null), port:0
[2020-02-16 09:22:09.525186] W [rpc-clnt.c:1753:rpc_clnt_submit] 6-vol_50cfefdda6788f3c240dbc503591c71c-client-1: error returned while attempting to connect to host:(null), port:0
[2020-02-16 09:22:09.526273] W [rpc-clnt.c:1753:rpc_clnt_submit] 6-vol_50cfefdda6788f3c240dbc503591c71c-client-1: error returned while attempting to connect to host:(null), port:0
```

### 解决方案

[gluster官网BUG补丁](<https://review.gluster.org/#/c/glusterfs/+/22289/>),  已经合并到master分支

[Code Review](<https://review.gluster.org/#/c/glusterfs/+/22289/2/rpc/rpc-lib/src/rpc-clnt.c@1696>)

[Redhat问题描述](<https://bugzilla.redhat.com/show_bug.cgi?id=1676546>)，问题发生在 glusterfs 4.1.5版本，但是master分支已经恢复

This patch avoids printing of "error returned while attempting to" unless loglevel is set to GF_LOG_DEBUG.

We don't need to change this message to GF_DEBUG, log is expected if the client is not able to connect with the server. As per logs are showing in bugzilla it seems host_name and port both are NULL so client was not able to connect with server. error returned while attempting to connect to host:(null), port:0