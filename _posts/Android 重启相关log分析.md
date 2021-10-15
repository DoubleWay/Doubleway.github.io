## Android 重启相关log分析

##### 重启后的kernel.log

在kernel.log里搜索androidboot.mode，如果没有则说明是noemal mode，正常开机。如果有`panic`，则说明可能是kernel层造成的重启

##### 还可以看重启后的last_kmsg.log

此文件记录了重启前的kernel log，搜索`panic`确定问题发生在上层还是底层，如果有panic相关信息一般都是kernel造成的问题，如果有Restarting system则是上层造成的。

```
reboot: Restarting system with command 'userrequested'
```

