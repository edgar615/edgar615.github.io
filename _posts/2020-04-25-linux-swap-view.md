---
layout: post
title: 查看哪些进程在swap
date: 2020-04-25
categories:
    - linux
comments: true
permalink: linux-swap-view.html
---



```sh
#!/bin/bash  
# Get current swap usage for all running processes  
# writted by xly  
  
function getswap {  
SUM=0  
OVERALL=0  
for DIR in `find /proc/ -maxdepth 1 -type d | egrep "^/proc/[0-9]"` ; do  
PID=`echo $DIR | cut -d / -f 3`  
PROGNAME=`ps -p $PID -o comm --no-headers`  
for SWAP in `grep Swap $DIR/smaps 2>/dev/null| awk '{ print $2 }'`  
do  
let SUM=$SUM+$SWAP  
done  
echo "PID=$PID - Swap used: $SUM - ($PROGNAME )"  
let OVERALL=$OVERALL+$SUM  
SUM=0  
  
done  
echo "Overall swap used: $OVERALL"  
}  
  
getswap  
#getswap|egrep -v "Swap used: 0"
```

