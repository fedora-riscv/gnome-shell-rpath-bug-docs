# gnome-shell v44.2 在 RISC-V 平台下 segfault 的问题发现及修复
起初是发现 Fedora 38 RISC-V 在执行 `gnome-shell --list-modes` 时出现 segfault 的情况，便对这一问题展开了调查

## 过程
在干净的 Fedora 38 RISC-V 上安装 gnome-shell(https://openkoji.iscas.ac.cn/koji/buildinfo?buildID=51424)，并测试 `gnome-shell --list-modes` 确实有问题，随后下载了该包的 debugsource 和 debuginfo 并使用 gdb 分析

segfault的调用堆栈如下图
![image](https://github.com/fedora-riscv/gnome-shell-rpath-bug-docs/assets/5274559/00cfa3ae-4afa-48b1-bdb3-534e0fdb9bf2)

查看后发现 strstr 传入了一个空指针，遂追踪空指针何时被传入，追踪到 `shell_introspection_init()` 时发现找不到 `g_strsplit()` 函数，后来发现此函数是在 `maybe_add_rpath_introspection_paths()` 里调用到的，而这个函数应该是被 gcc 优化为 `shell_introspection_init()` 的内联函数了

可以看到 

[https://github.com/GNOME/gnome-shell/blame/0eec6fea6913aae84724391224990091a215059f/src/main.c#L156-L159](https://github.com/GNOME/gnome-shell/blame/0eec6fea6913aae84724391224990091a215059f/src/main.c#L156-L159)

此处调用了 `g_strsplit()`，而 strtab 出自上面的 for 循环

在 gdb 里打印 strtab 的值会发现该值为 0x4d50，不像是一个地址，同时在 x86 下的 gnome-shell 测试中，strtab 的值就是一个正确的指针，那么可以推断出是 strtab 指向了一个不存在的地址导致后续问题

搜索过程中发现 gnome-shell 的 gitlab 中已经有人提出了 [相关issue](https://gitlab.gnome.org/GNOME/gnome-shell/-/issues/6528)

根据他们的说法，glibc 对于 MIPS 和 RISC-V 的 .dynamic 段因为是只读的默认没有开启 reallocate，导致获取的地址仅仅只是 offset 而非实际地址，因此在 [该commit中](https://gitlab.gnome.org/GNOME/gnome-shell/-/commit/667ff703864e4c427234444bbb98d6ed82c1c4e5) 简单判断了 strtab 是否小于 link_map->l_addr，如果是，则相加，相当于手动偏移地址

在应用该补丁后问题得到解决

感谢 @U2FsdGVkX1 在解决此问题时提供的帮助

