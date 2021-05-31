## 测试 clone(), join() 和线程库

1. 将 Syscall_test 目录下的 clonetest.c, jointest.c 和 kthreads.c 复制到xv6-user目录下

2. 修改Makefile
```
UPROGS=\
    $U/_init\
    $U/_sh\
    $U/_cat\
    ...
    $U/_clonetest\      # Don't ignore the leading '_'
    $U/_jointest\
    $U/_kthreads\
```
3. 在根目录下运行如下命令，编译用户程序
```bash
make userprogs
```

4. 重新将编译好的文件镜像烧录在sd卡中
```bash
make sdcard sd="sdcard path"
```