---
layout: post
category : CMake
title: 用 CMake 的一点经验
description : 记录使用 CMake 遇到的一些问题和解决办法
tags : [CMake]
---

公司项目的 build 有两个版本： 静态链接(static)和动态链接(dll)版本。 dll 版本增量 build 快，开发效率高。 我们可以在开发机器上 build, 然后把 build 文件夹 mount 到 cluster 某个 node 上，再在 cluster 上 debug， 很方便。

但由于最近从 `GNU make` 迁移到了 `CMake`, 这种方法行不通了。下面记录的就是怎样解决这个问题的。


原因
---
有两个：

 1. 项目中各个模块输出的 so 文件都是在各个模块的文件夹下，不是在同一个地方，用 `LD_LIBRARY_PATH` 去把所有的模块输出文件路径加进来不现实。

 2. 虽然把所有用到的 3rd party libs 集中放在了一个地方并通过 `LD_LIBRARY_PATH` 加进来，但 `ldd` 还是提示说某些 3rd party lib 找不到，如：

        /home/miliao/dll_third_libs/loki/2005.11.16/lib/libloki.so => not found

解决办法
---
#### 模块输出的 so 文件
可以通过设置 CMake 变量 `CMAKE_LIBRARY_OUTPUT_DIRECTORY` 把输出的 so 文件集中放在一起：

    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/libs)

再通过 `LD_LIBRARY_PATH` 把这个路径加上就行了。


#### 某些 3rd party lib 找不到
为了使用 3rd party lib， 我们用了两种方法：

 1. 通过 `find_package(XXX)` 来找，前提是 CMake 官方提供有 `FindXXX` 模块，否则要用到下面的方法
 2. 手动添加：

        add_library(XXX SHARED IMPORTED)
        set_property(TARGET XXX PROPERTY INTERFACE_INCLUDE_DIRECTORIES /path/to/include)
        set_property(TARGET XXX PROPERTY IMPORTED_LOCATION /path/to/libXXX.so)

所有通过第一种方法和部分第二种方法用到的 3rd party lib 在运行时都能从 `LD_LIBRARY_PATH` 设置的路径中正确找到，只有少数几个用第二种方法的找不到。

能找到的 3rd party lib 在用 `ldd` 命令查看时的输出都是这种格式：

    libchartdir.so.5.1 => /group/DEV/GUI/yyuan/dll_third_libs/libchartdir.so.5.1 (0x00007fec6c4eb000)

和上面找不到 `libloki.so` 的错误信息比较下可以看到这里有个差别就是能找到的都是只有 lib name，而不是 full path。

##### NEEDED & SONAME
可以通过 `readelf -d` 或者 `objdump -p` 命令查看 ELF 文件的 dynamic section。其中 `NEEDED` 就是这个文件所依赖的所有其他文件， `SONAME` 只用在 compile/link，运行期不需要用到。

我们看下 `libchartdir.so.5.1` 和 `libloki.so` 这两个文件的 dynamic section:

`libchartdir.so.5.1`:

    Dynamic section at offset 0x4138c0 contains 25 entries:
      Tag        Type                         Name/Value
     0x0000000000000001 (NEEDED)             Shared library: [libdl.so.2]
     0x0000000000000001 (NEEDED)             Shared library: [libm.so.6]
     0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
     0x000000000000000e (SONAME)             Library soname: [libchartdir.so.5.1]
     0x000000000000000c (INIT)               0xb7e18
     0x000000000000000d (FINI)               0x30e998
     0x0000000000000004 (HASH)               0x158
     0x0000000000000005 (STRTAB)             0x70e0
     0x0000000000000006 (SYMTAB)             0x1800
     0x000000000000000a (STRSZ)              19753 (bytes)
     0x000000000000000b (SYMENT)             24 (bytes)
     0x0000000000000003 (PLTGOT)             0x513af8
     0x0000000000000002 (PLTRELSZ)           2448 (bytes)
     0x0000000000000014 (PLTREL)             RELA
     0x0000000000000017 (JMPREL)             0xb7488
     0x0000000000000007 (RELA)               0xc620
     0x0000000000000008 (RELASZ)             700008 (bytes)
     0x0000000000000009 (RELAENT)            24 (bytes)
     0x000000006ffffffc (VERDEF)             0xc578
     0x000000006ffffffd (VERDEFNUM)          2
     0x000000006ffffffe (VERNEED)            0xc5b0
     0x000000006fffffff (VERNEEDNUM)         3
     0x000000006ffffff0 (VERSYM)             0xbe0a
     0x000000006ffffff9 (RELACOUNT)          29157
     0x0000000000000000 (NULL)               0x0

`libloki.so`:

    Dynamic section at offset 0x4470 contains 23 entries:
      Tag        Type                         Name/Value
     0x0000000000000001 (NEEDED)             Shared library: [libstdc++.so.6]
     0x0000000000000001 (NEEDED)             Shared library: [libm.so.6]
     0x0000000000000001 (NEEDED)             Shared library: [libgcc_s.so.1]
     0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
     0x000000000000000c (INIT)               0x2190
     0x000000000000000d (FINI)               0x3b38
     0x0000000000000004 (HASH)               0x158
     0x0000000000000005 (STRTAB)             0xdf0
     0x0000000000000006 (SYMTAB)             0x478
     0x000000000000000a (STRSZ)              3148 (bytes)
     0x000000000000000b (SYMENT)             24 (bytes)
     0x0000000000000003 (PLTGOT)             0x104670
     0x0000000000000002 (PLTRELSZ)           960 (bytes)
     0x0000000000000014 (PLTREL)             RELA
     0x0000000000000017 (JMPREL)             0x1dd0
     0x0000000000000007 (RELA)               0x1b78
     0x0000000000000008 (RELASZ)             600 (bytes)
     0x0000000000000009 (RELAENT)            24 (bytes)
     0x000000006ffffffe (VERNEED)            0x1b08
     0x000000006fffffff (VERNEEDNUM)         3
     0x000000006ffffff0 (VERSYM)             0x1a3c
     0x000000006ffffff9 (RELACOUNT)          4
     0x0000000000000000 (NULL)               0x0

可以看到这两个 so 文件有个差别就是 `libloki.so` 缺少了 `SONAME`。

CMake 在用 `add_library(XXX SHARED IMPORTED)` 并通过 `IMPORTED_LOCATION` 指定 lib full path 导入 lib 时会看这个 lib 有没有 `SONAME`。如果有的话就把 lib full path 截断并通过 `-L/path/to/libXXX -lXXX`来链接，如果没有的话就会用全路径 `-l/path/to/libXXX.so` 来链接。这会导致在链接生成的文件里面 `NEEDED` 字段一个是 lib name 一个是 lib full path。

可以查看最后链接生成的可执行文件的 dynamic section来看下：

    0x0000000000000001 (NEEDED)             Shared library: [libchartdir.so.5.1]
    0x0000000000000001 (NEEDED)             Shared library: [/home/miliao/dll_third_libs/loki/2005.11.16/lib/libloki.so]

全路径的 `libloki.so` 显然是没法在我们设置的 `LD_LIBRARY_PATH` 里面找到的，所以会报个 `not found` 的错误。

解决办法就是设置 `IMPORTED_NO_SONAME` 变量，显式的告诉 CMake 始终都把 lib full path 截断。以上面的 `libloki.so` 为例：

    set_property(TARGET loki PROPERTY IMPORTED_NO_SONAME 1)

其他
---
#### `ld.so` 查找 lib 的顺序
可以通过 `man ld.so` 查看，这里简单列下：

 1. `RPATH`： 在文件的 dynamic section
 2. `LD_LIBRARY_PATH`: 环境变量，比较常用
 3. `RUNPATH`： 在文件的 dynamic section，比较少用
 4. `/etc/ld.so.cache`
 5. 默认路径 `/lib` 和 `/usr/lib`

#### `ldd`
`ldd` 是个 bash script，是对 `ld.so/ld-linux.so` 的封装。

    [yyuan@bcnode060 ~]$ bash -x ldd /bin/ls
    + TEXTDOMAIN=libc
    + TEXTDOMAINDIR=/usr/share/locale
    + RTLDLIST='/lib/ld-linux.so.2 /lib64/ld-linux-x86-64.so.2'
    + warn=
    + bind_now=
    + verbose=
    + test 1 -gt 0
    + case "$1" in
    + break
    + add_env='LD_TRACE_LOADED_OBJECTS=1 LD_WARN= LD_BIND_NOW='
    + add_env='LD_TRACE_LOADED_OBJECTS=1 LD_WARN= LD_BIND_NOW= LD_LIBRARY_VERSION=$verify_out'
    + add_env='LD_TRACE_LOADED_OBJECTS=1 LD_WARN= LD_BIND_NOW= LD_LIBRARY_VERSION=$verify_out LD_VERBOSE='
    + test '' = yes
    + set -o pipefail
    + case $# in
    + single_file=t
    + result=0
    + for file in '"$@"'
    + test t = t
    + case $file in
    + :
    + test '!' -e /bin/ls
    + test '!' -f /bin/ls
    + test -r /bin/ls
    + test -x /bin/ls
    + RTLD=
    + ret=1
    + for rtld in '${RTLDLIST}'
    + test -x /lib/ld-linux.so.2
    ++ /lib/ld-linux.so.2 --verify /bin/ls
    + verify_out=
    + ret=1
    + case $ret in
    + for rtld in '${RTLDLIST}'
    + test -x /lib64/ld-linux-x86-64.so.2
    ++ /lib64/ld-linux-x86-64.so.2 --verify /bin/ls
    + verify_out=
    + ret=0
    + case $ret in
    + RTLD=/lib64/ld-linux-x86-64.so.2
    + break
    + case $ret in
    + try_trace /lib64/ld-linux-x86-64.so.2 /bin/ls
    + eval LD_TRACE_LOADED_OBJECTS=1 LD_WARN= LD_BIND_NOW= 'LD_LIBRARY_VERSION=$verify_out' LD_VERBOSE= '"$@"'
    + cat
    ++ LD_TRACE_LOADED_OBJECTS=1
    ++ LD_WARN=
    ++ LD_BIND_NOW=
    ++ LD_LIBRARY_VERSION=
    ++ LD_VERBOSE=
    ++ /lib64/ld-linux-x86-64.so.2 /bin/ls
        linux-vdso.so.1 =>  (0x00007fff84ed6000)
        libselinux.so.1 => /lib64/libselinux.so.1 (0x0000003001400000)
        librt.so.1 => /lib64/librt.so.1 (0x0000003000c00000)
        libcap.so.2 => /lib64/libcap.so.2 (0x0000003002c00000)
        libacl.so.1 => /lib64/libacl.so.1 (0x000000300e400000)
        libc.so.6 => /lib64/libc.so.6 (0x0000003fffc00000)
        libdl.so.2 => /lib64/libdl.so.2 (0x0000003fff800000)
        /lib64/ld-linux-x86-64.so.2 (0x0000003fff400000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x0000003000000000)
        libattr.so.1 => /lib64/libattr.so.1 (0x000000300d200000)
    + exit 0

可以看到 `ldd` 其实就是设置了一些 `LD_XXX` 环境变量然后调用 `ld-linux-x86-64.so.2` 来真正执行。 感兴趣的可以通过 `man ld.so` 查看下这些环境变量的作用。
