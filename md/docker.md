# Docker不是虚拟机

# 虚拟化的种类和层级

1. cpu虚拟化：可以模拟不同CPU，例如bochs
2. 完全虚拟化：只能模拟同样CPU，但是可以执行不同系统，例如vmware
3. 半虚拟化：guest必须打补丁，例如Xen
4. 硬件虚拟化：可以当作获得硬件加速的完全虚拟化
5. 系统虚拟化：host和guest共享一样的内核，例如Openvz
6. 语言沙盒：只能在语言的范围内使用

---

虚拟化的级别越偏底层，速度越慢，用户越难察觉到虚拟化的存在。

虚拟化的级别越偏上层，速度越快，用户越容易感知。

1. cpu虚拟化和完全虚拟化时，用户几乎可以不察觉到虚拟化的存在
2. 半虚拟化时，guest内核必须存在补丁
3. 系统虚拟化时，用户不能控制自己的内核
4. 语言沙盒时，用户没有使用api的自由

# docker的实现结构

* docker
  * lxc
    * namespace: 仅沙盒隔离，不限制资源。
	* cgroup: 仅限制资源，不沙盒隔离。
  * aufs
  * image管理

当然，还有很多细节的东西，里面就不一一列举了。例如veth。

# docker不是虚拟机

docker不是虚拟机，因为lxc已经是虚拟机。

如果两者功能一样，那么docker就没有存在的必要。

你可以把docker当虚拟机用，但是当虚拟机用的话，他的完备程度远远不及现在的种种虚拟机。

相比之下，就会觉得很不好用。这不是docker的错，只能说被不正确的使用了。

## docker是什么

docker就是环境。

docker实际上只做了一件事情——镜像管理。负责将可执行的镜像导入导出，在不同设备上迁移。

---

原本我们发布软件有两种方法，源码发布和二进制发布。二进制发布又有两种方案，静态链接和动态链接。

最早的时候，我们发布软件都喜欢动态链接，因为小。

但是随着网络和存储的升级，软件越来越喜欢静态链接，或者把动态库打包到发布里。

因为系统情况越来越复杂，依赖关系一旦出错，系统就无法启动。

将这个思路推到极限，就是虚拟机发布。早些年有人发过一些Oracle的linux安装镜像，算的上是先驱。

因为Oracle早些年的安装程序很难用，对系统的依赖复杂。公司做测试用装一套Oracle还不够麻烦的。

相比起来，下载一个虚拟机直接跑起来就可以用就方便了很多。即使性能差一些，测试而已也不是特别在意。

docker再进了一步。不但提供一个镜像，可以在系统间方便的迁移。而且连镜像的升级都能做掉。

更爽的是，升级只用传输差量数据。

当然，有好处就有牺牲。

## docker的镜像是只读的

其实不是，docker的镜像当然可以写入。但是写的时候有几个问题。

1. 如果对镜像进行写入，aufs会将原始文件复制一次，再进行写入。这样性能比较低。
2. 更直接的问题是，一旦对镜像做了写入，就无法从docker这里获得更新支持——docker不能将你的写入和上游的更新合并。因此，整个系统就退化成了一个完全的虚拟机。

所以，我个人认为，docker的镜像本身应当是只读的——如同EC2里面一样。数据的写入应当通过远程文件系统或者数据库服务来解决。

# vagrant

提到镜像管理，我们可以提一下同样属于镜像管理的一个软件——vagrant。

1. 可以将vbox的镜像打包导入导出
2. 提供了一个cloud，允许镜像的分享/更新

## 为什么vagrant不如docker出名

* 快，系统级虚拟化使得docker的虚拟化开销降低到百分级别以下。
* 可以在虚拟机内使用的虚拟机，例如云主机内。
* 资源调度灵活，不需要将资源预先划定给不同的实例，在不同资源的机器上也不用调整参数。

# 典型用例

## 编译系统/打包系统/集成测试环境

典型的搭建一次，执行一次，销毁一次。不需要对image做更改（准确说的需要做更改，但是不需要保存）。

## 公司内部应用

在IaaS的比拼中，以Openvz为代表的系统化虚拟化方案几乎完败于完全虚拟化/半虚拟化系列技术。就我和朋友的讨论，这里面最主要的因素在于。完全虚拟化技术可以比较好的隔离实例和实例间的资源使用，而系统虚拟化技术更偏向于将资源充分利用。这使得系统虚拟化更容易超售。

然而，在公司内部应用中，这一缺陷就变成了优势。企业的诸多系统，只要在同一个优先级，其可用性应当是一致的。几个联动系统中，一个资源不足陷于濒死的情况下，保持其他几个系统资源充足并无意义。而且总资源是否足够应当是得到充分保证的事情，企业自己“超售”自己的资源，使得业务系统陷入运行缓慢的境地一点意义都没有。

---

因此，系统虚拟化可以为企业级云计算提供可以灵活调度的资源，和非常低的额外开销。

当然，云计算在企业化中原本就面临一些问题。原本提供软-硬件统一解决方案的集成商，需要如何重新组织解决方案。如何协调节约资源和高性能，高可用。云计算在企业级应用中还有很长的路要走。

# 短板

* 太新。目前成功案例还是不足，而且围绕docker的工具链还不完备。
* 适用范围比较窄。需求需要集中在“环境迁移”领域，而且image本身不应被写入。
* 生不逢时。rvm和virtualenv已经在前面了。

# FAQ