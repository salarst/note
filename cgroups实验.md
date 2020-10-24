### 实验目的
- 验证CPU限制成功
- 验证应用超出内存限制后会报OOM

### 安装cgroup命令行工具
	yum -y install libcgroup-tools

### cpu测试脚本
```
#!/usr/bin/python
i = 0
while True:
    i += 1
```

```
./cpu.py
```
top 查看当前cpu.py的cup使用情况
![](https://github.com/salarst/note/blob/master/img/cpu_usage.png)
可看出cpu使用率为100%

利用cgroup的cpu资源限制，使得该脚本cpu使用率为20%
```
mkdir /sys/fs/cgroup/cpu/test
cd /sys/fs/cgroup/cpu/test
echo 20000 > cpu.cfs_quota_us  //1个cpu为100000,此值配置在cpu.cfs_period_us里，此处20000/100000=20%
echo 24289 > tasks
```
再次top查看

### 测试内存脚本
```
#!/usr/bin/python
import time
   #表示1G
size = 1*1024*1024*1024 
   #1个空间为1B，此处写10G
while True:
    a = ' ' * size
    time.sleep(1)
```
```
./mem.py
```
top查看,可发现内存在不停的增加
利用memeory子系统，限制内存为10m
```
cd /sys/fs/cgroup/memory
mkdir test
echo 10m > memory.limit_bytes
echo 10m > memory.memsw.limit_bytes  //由于本机有swap分区，所以要限制使用swap的量，不然mem.py还可以使用swap空间
echo 27163 > tasks
```
配置完后，mem.py马上会报错退出
查看message日志信息如下
```
[root@mysql2 tmp]# tailf /var/log/messages
Oct 24 16:38:12 mysql2 kernel: [<ffffffffadf89778>] page_fault+0x28/0x30
Oct 24 16:38:12 mysql2 kernel: Task in /test killed as a result of limit of /test
Oct 24 16:38:12 mysql2 kernel: memory: usage 10240kB, limit 10240kB, failcnt 2168724
Oct 24 16:38:12 mysql2 kernel: memory+swap: usage 10240kB, limit 10240kB, failcnt 0
Oct 24 16:38:12 mysql2 kernel: kmem: usage 0kB, limit 9007199254740988kB, failcnt 0
Oct 24 16:38:12 mysql2 kernel: Memory cgroup stats for /test: cache:0KB rss:10240KB rss_huge:8192KB mapped_file:0KB swap:0KB inactive_anon:10220KB active_anon:0KB inactive_file:0KB active_file:0KB unevictable:0KB
Oct 24 16:38:12 mysql2 kernel: [ pid ]   uid  tgid total_vm      rss nr_ptes swapents oom_score_adj name
Oct 24 16:38:12 mysql2 kernel: [27163]     0 27163   555677   265808     535        0             0 mem.py
Oct 24 16:38:12 mysql2 kernel: Memory cgroup out of memory: Kill process 27163 (mem.py) score 100919 or sacrifice child
Oct 24 16:38:12 mysql2 kernel: Killed process 27163 (mem.py), UID 0, total-vm:2222708kB, anon-rss:1061176kB, file-rss:2056kB, shmem-rss:0kB
```
