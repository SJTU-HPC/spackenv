Spackenv (SPACK ENvironment) https://git.hpc.sjtu.edu.cn/devops/spackenv 是用于配置Spack软件包管理器的脚本。
这个脚本适用于系统软件包管理员、普通Spack用户和Spack开发者，通过编写或复用已有的YAML格式的环境文件，使用者可在不同的Spack环境间切换。
这些Spack环境可共享编译器等基础软件，但由于使用不的安装路径和模块命名方式，这些环境可以无冲突共存。
Spackenv 可根据筛选条件为部分面向最终用户的软件生成 Environment Modules ，集群用户无需了解Spack知识也无需配置Spack环境就能调用软件，因此是Spackenv是Pi集群软件部署的首选方式。

使用流程
========

首先，需要由系统管理员和软件包管理员使用Spackenv完成冷启动。
然后，其他Spack用户可在冷启动配置文件基础上修改，构建自己的Spack环境。
对于只需要调用软件完成计算任务的用户，可直接载入 Environment Modules 进行计算，无需配置Spack。

软件包管理员为Spack添加编译器(冷启动)
-------------------------------------

在冷启动中，由管理员为软件包管理员创建软件安装目录，然后软件包管理员调用spackenv中的脚本令环境生效，安装gcc、intel编译器后，将编译器路径写入配置文件中供后续使用。

首先，选定如下几个关键路径，由系统管理员创建目录并将目录属主改为软件包管理员。
对于软件包管理员，以CascadeLake为例，推荐使用如下值。

+---------------------+------------------------------------------------------------+---------------------------------------------+
| 配置变量            |  备注                                                      | 推荐值                                      |
+=====================+============================================================+=============================================+
| spack_bin_root      | Spack项目的目录位置                                        | ``/lustre/spack``                           |
+---------------------+------------------------------------------------------------+---------------------------------------------+
| spack_install_root  | Spack软件包安装根目录(建议不同代CPU的目录分开)             | ``/lustre/opt/cascadelake``                 |
+---------------------+------------------------------------------------------------+---------------------------------------------+
| spack_modules_root  | Spack Environment Module写入位置(建议不同代CPU的目录分开)  | ``/lustre/share/spack/modules/cascadelake`` |
+---------------------+------------------------------------------------------------+---------------------------------------------+

使用系统管理员权限创建目录，并更改属主为 ``rpm``::

  # export spack_bin_root=/lustre/spack; export spack_install_root=/lustre/opt/cascadelake; export spack_modules_root=/lustre/share/spack/modules/cascadelake
  # export pkg_mgn=rpm
  # mkdir -p $spack_bin; mkdir -p $spack_install_root; mkdir -p $spack_modules_root
  # chown -R $pkg_mgn $spack_bin; chown -R $pkg_mgn $spack_install_root; chown -R $pkg_mgn $spack_modules_root

由系统管理员克隆spack代码，并更改属主为 ``rpm``::

  # cd /lustre; git clone https://github.com/spack/spack.git
  # export pkg_mgn=rpm
  # chown -R $pkg_mgn spack/

在软件包管理员账户下克隆spackenv项目::

  $ git clone https://github.com/SJTU-HPC/spackenv.git

在项目的env中复制空的示例环境文件 ``emtpy.yaml`` ::

  $ cd env
  $ cp emtpy.yaml pi2-system-cascadelake.yaml

在新的环境文件中，根据提示，将如上的配置值赋予相应的变量，头几行内容如下::

  ---
  spack_bin_root:         /lustre/spack
  spack_install_root:     /lustre/opt/cascadelake
  spack_modules_root:     /lustre/share/spack/modules/cascadelake

运行 ``install`` 写入Spack配置文件::

  $ ./install --env env/pi2-system-cascadelake.yaml

在软件包管理员 ``~/.zshrc`` 或 ``~/.bashrc`` 中增加配置，制定 ``spack`` 位置，环境变量 ``SPACK_ROOT`` 应该与 ``spack_bin_root`` 一致::

    export SPACK_ROOT=/lustre/spack
    source $SPACK_ROOT/share/spack/setup-env.sh
 
重新登录，检查spack是否可用::

  $ which spack 

使用spack安装gcc, pgi, intel编译器套件::

  $ spack install gcc@8.3.0 languages=ada,c,c++,fortran %gcc@4.8.5
  $ spack install pgi@19.4+nvidia+single~network %gcc@4.8.5
  $ rm -rf ~/intel; spack install intel@19.0.4 %gcc@4.8.5
  $ rm -rf ~/intel; spack install intel-parallel-studio@cluster.2018.3+advisor+clck+daal+inspector+ipp+itac+mkl+mpi+tbb+vtune %gcc@4.8.5 threads=openmp
  $ rm -rf ~/intel; spack install intel-parallel-studio@cluster.2019.4+advisor+clck+daal+inspector+ipp+itac+mkl+mpi+tbb+vtune %gcc@4.8.5 threads=openmp

注意，Intel软件在安装时会检查计算机，若发现已安装的组件会终止安装过程，因此在使用spack安装Intel软件前需要删除 ``~/intel`` 目录。
部分Intel软件需要软件授权，可使用EDU邮箱在 https://registrationcenter.intel.com/regcensec/login.aspx 申请，或使用 https://git.hpc.sjtu.edu.cn/devops/puppet-environments/blob/pi2/data/roles/compute.yaml 提供的 ``intel2019.lic`` 授权文件。

安装完编译器后，需要将编译器位置写入环境文件。
以intel编译器为例，查找软件包的安装位置::

  $ spack location -i intel@19.0.4
  /lustre/opt/cascadelake/linux-centos7-x86_64/gcc-4.8.5/intel-19.0.4-zxtvkrsgqvhud4og62e55kwgqvhy26d3

在环境配置文件 ``pi2-system.yaml`` 的 ``compilers`` 域增加intel编译器入口，内容如下::

  - compiler:
      environment: {}
      extra_rpaths: []
      flags: {}
      modules: []
      operating_system: centos7
      paths:
        cc:  /lustre/opt/cascadelake/linux-centos7-x86_64/gcc-4.8.5/intel-19.0.4-zxtvkrsgqvhud4og62e55kwgqvhy26d3/compilers_and_libraries_2019.3.199/linux/bin/intel64/icc
        cxx: /lustre/opt/cascadelake/linux-centos7-x86_64/gcc-4.8.5/intel-19.0.4-zxtvkrsgqvhud4og62e55kwgqvhy26d3/compilers_and_libraries_2019.3.199/linux/bin/intel64/icpc
        f77: /lustre/opt/cascadelake/linux-centos7-x86_64/gcc-4.8.5/intel-19.0.4-zxtvkrsgqvhud4og62e55kwgqvhy26d3/compilers_and_libraries_2019.3.199/linux/bin/intel64/ifort
        fc:  /lustre/opt/cascadelake/linux-centos7-x86_64/gcc-4.8.5/intel-19.0.4-zxtvkrsgqvhud4og62e55kwgqvhy26d3/compilers_and_libraries_2019.3.199/linux/bin/intel64/ifort
      spec: intel@19.0.3
      target: x86_64

类似地，可继续将Spack安装的其他编译器也加入环境配置文件中，然后重新生成配置文件::

  $ ./install --env env/pi2-system-cascadelake.yaml

确认编译器已经能被Spack识别，并能编译库::

  $ spack compilers
  ...
  intel@19.0.4
  ...
  $ spack install zlib %intel@19.0.4

完成上述冷启动流程后， ``pi2-system-cascadelake.yaml`` 是一个带有若干编译器的环境，可作为后续环境配置的基础。

调用Spackenv构建的Spack环境
==========================

软件包管理员
------------

软件包管理员(``rpm``)可继续使用 ``pi2-system-cascadelake.yaml`` 环境，安装供所有用户使用的软件。
以octave为例，如下流程切换到了供软件包管理员使用的Spack环境、安装octave、生成Octave模块::

  $ ./install --env env/pi2-system-cascadelake.yaml
  $ spack install octave+qt+llvm %gcc@8.3.0 ^openblas ^llvm@7.0.1
  $ spack module tcl refresh --delete-tree -y

更加推荐的方式是，将安装指令 ``octave+qt+llvm %gcc@8.3.0 ^openblas threads=openmp ^llvm@7.0.1`` 加入到环境配置文件的 ``install`` 域，内容如下::

  - 'spack spec octave+qt+llvm %gcc@8.3.0 ^openblas ^llvm@7.0.1'

然后调用Spackenv脚本完成软件安装安装和模块更新任务，在配置文件中可以一次加入多个软件包::

  $ ./install --env env/pi2-system-cascadelake.yaml  --install --refresh

**为避免误操作，软件包管理员在为系统添加软件前，需要以另外一个普通用户的身份进行软件包安装测试。**

需要使用Spack的集群用户
-----------------------

为避免与系统安装的软件冲突，普通该用户使用的安装位置和模块位置做了如下修改：

+---------------------+------------------------------------------------------------+---------------------------------------------+
| 配置变量            |  备注                                                      | 推荐值                                      |
+=====================+============================================================+=============================================+
| spack_bin_root      | Spack项目的目录位置                                        | ``~/spack``                                 |
+---------------------+------------------------------------------------------------+---------------------------------------------+
| spack_install_root  | Spack软件包安装根目录(建议不同代CPU的目录分开)             | ``~/opt/cascadelake``                       |
+---------------------+------------------------------------------------------------+---------------------------------------------+
| spack_modules_root  | Spack Environment Module写入位置(建议不同代CPU的目录分开)  | ``~/share/spack/modules/cascadelake``       |
+---------------------+------------------------------------------------------------+---------------------------------------------+

克隆Spack项目::

  $ git clone https://github.com/spack/spack.git

克隆Spackenv项目::

  $ git clone https://git.hpc.sjtu.edu.cn/devops/spackenv.git

在 ``~/.zshrc`` 或 ``~/.bashrc`` 中增加配置，指定 ``spack`` 位置，环境变量 ``SPACK_ROOT`` 应该与 ``spack_bin_root`` 一致::

    export SPACK_ROOT=$HOME/spack
    source $SPACK_ROOT/share/spack/setup-env.sh

可以从 ``pi2-system.yaml`` 配置文件开始，定制自己的环境::

  $ cp env/pi2-system-cascadelake.yaml env/pi2-user.yaml
  $ ./install --env env/pi2-user.yaml

特别需要注意的是，为避免模块名字重复，需要在配置文件中修改命名方式::

  naming_scheme: '{name}-{version}-{compiler.name}-{compiler.version}'

通常Spackenv仓库自带的 ``pi2-user.yaml`` 已经为普通用户做了一些定制，可直接使用::

  $ ./install --env env/pi2-user-cascadelake.yaml

与软件包管理员类似，普通用户可以直接使用 ``spack`` 或者 ``./install`` 安装新软件。

Spack软件包开发者
-----------------

Spack软件包开发者通常在自己的Spack代码分支上工作，不使用全局的Spack：

+---------------------+------------------------------------------------------------+---------------------------------------------+
| 配置变量            |  备注                                                      | 推荐值                                      |
+=====================+============================================================+=============================================+
| spack_bin_root      | Spack项目的目录位置                                        | ``~/spack``                           |
+---------------------+------------------------------------------------------------+---------------------------------------------+
| spack_install_root  | Spack软件包安装根目录(建议不同代CPU的目录分开)             | ``~/opt/cascadelake``                       |
+---------------------+------------------------------------------------------------+---------------------------------------------+
| spack_modules_root  | Spack Environment Module写入位置(建议不同代CPU的目录分开)  | ``~/share/spack/modules/cascadelake``       |
+---------------------+------------------------------------------------------------+---------------------------------------------+

定制自己的环境后，就可以在这个环境中进行Spack开发和测试::

  $ cp env/pi2-system-cascadelake.yaml env/pi2-dev-cascadelake.yaml
  $ ./install --env env/pi2-dev-cascadelake.yaml

通过Environment Modules使用Spackenv安装的软件
=============================================

自动生成Environment Modules是Spack的亮点功能，可免去管理员手工编写模块文件的工作量。
对需要调用软件完成计算任务的普通集群用户，调用Modules也免去了学习Spack机制的负担。

Spack生成的Environment Modules，其路径在 ``/etc/profile.d/modulepath.sh`` 中制定，在 Pi 2.0 上内容如下，其中路径 ``/lustre/share/spack/modules/cascadelake/linux-centos7-x86-64`` 下存放的就是Spack生成的模块文件。

  #!/bin/bash
  # File managed by Puppet
  
  export PLATFORM=cascadelake
  export MODULEPATH=/lustre/usr/modulefiles:/lustre/share/spack/modules/$PLATFORM/linux-centos7-x86_64:$MODULEPATH

模块文件根据配置文件中的 ``naming_schmeme`` 进行命名，以系统使用的 ``'{name}/{version}-{compiler.name}-{compiler.version}'`` 为例，下面几个例子展示了这种 ``软件名/软件版本-编译器-编译器版本`` 命名规范::

  samtools/1.9-intel-19.0.4
  cuda/10.0.130-gcc-5.4.0
  intel/19.0.4-gcc-4.8.5

受限于Spack的软件构建模型，这样的组合不一定反映真实情况。譬如，cuda-10.0.130并不是gcc-5.4.0构建生成的，这里表示两者可以一起兼容工作。intel-19.0.4编译器也不是gcc-4.8.5构建生成的，只不过所有编译器在安装时，也需要指定一个“构建”它的编译器。

R和Python扩展模块比较特殊，安装后调用相应的R和Python模块就可以使用，不需要再加载具体扩展模块。

Spackenv其他操作
================

更新或替换软件包
----------------

1. 提前3个月告知用户旧软件下线；
2. 使用 ``spack unisntall`` 卸载旧版本软件；
3. 更新环境文件，使用新版本替换旧版本，运行 ``./install`` 部署新软件、更新模块文件；

Variant
-------

在配置文件的 ``packages`` 部分，可为软件包配置默认variants，避免在命令行输入复杂的variant指令。

模块黑名单
----------

在配置文件的 ``blacklist`` 部分，可指定不生成 Environment Modules 软件包列表，以减少暴露给用户的软件模块数量，或用于解决模块名字冲突的问题。

依赖包后缀
----------

为了让Environment Modules名字携带更多信息、尽量避免不同软件包的模块名冲突，可在配置文件的 ``modules`` -> ``tcl`` -> ``all`` -> ``suffixes`` 域增加以依赖包命名的后缀规则，譬如::

  suffixes:
    ^openblas: 'openblas'
    ^atlas: 'atlas'

加入以上配置后，链接到openblas和链接到atlas两个不同BLAS后端的octave编译实例，会生成不同Module名，从而避免名字冲突。

常见问题
========

模块名字冲突
------------

Environment Modules 模块名中只使用了软件名、版本、编译器、编译器版本，有可能出现名字冲突，可以尝试两种两种方法解决：

1. 修改编译选项重新编译，把名字冲突的软件(通常在variants上有细微差别) “归并”到同一个软件包。
2. 在blacklist中添加不希望生成模块的软件包，blacklist的过滤条件可以包括版本、编译选项、依赖包等。 

例如，运行 ``spack module tcl refresh -y`` 后提示有如下冲突::

  file: /lustre/share/spack/modules/sandybridge/linux-centos7-x86_64/htslib/1.9
  -intel-19.0.4
  spec: htslib@1.9%intel@19.0.4 arch=linux-centos7-x86_64
  spec: htslib@1.9%intel@19.0.4 arch=linux-centos7-x86_64

这两个htslib的名字、版本、所用编译器相同，生成的Environment Module名字相，导致了模块名字冲突。
需要将不希望生成的软件包加入到黑名单中，运行 ``spack find -dl htslib@1.9 %intel@19.0.4`` 查看着两个包的依赖库::

  ==> 2 installed packages
  -- linux-centos7-x86_64 / intel@19.0.4 --------------------------
  4zsx4qp    htslib@1.9
  z6ivwjm        ^bzip2@1.0.8
  plllndw        ^xz@5.2.4
  3ei36to        ^zlib@1.2.11
  
  6nmxiuu    htslib@1.9
  ymvfo2t        ^bzip2@1.0.6
  plllndw        ^xz@5.2.4
  3ei36to        ^zlib@1.2.11

决定不为Hash为 ``6nmxiuu`` 的htslib生成环境模块，在blacklist中加入这个库，然后重新运行 ``spack module tcl refresh -y``::

  - 'htslib@1.9 ^bzip2@1.0.6 %intel@19.0.4' 

软件包无法下载
--------------

因为种种原因，软件源码包可能无法在编译节点上下载。Pi 1.0 和 Pi 2.0 都部署了缓存服务节点，可手动将源码包上传到缓存节点目录下。
Pi 1.0 缓存服务器是 180.0.1.33 ，Pi 2.0 缓存服务器是 172.16.0.133 ，缓存的源代码都位于 ``/var/lib/www/spack.pi.sjtu.edu.cn/mirror/`` 目录下。
需要根据spack命令的错误提示 ``http://spack.pi.sjtu.edu.cn/mirror/xxx/xxx-xxx ``把下载的软件包重命名上传到相应目录下。

spack编译软件需要license的解决方法
--------------

spack编译的部分软件需要license才能正常使用，以pgi@19.4为例；
可通过官网下载，使用源码包进行手动编译，可以生成一个有时效的license::

   $ wget https://download.pgroup.com/secure/pgilinux-2019-194-x86-64.tar.gz?WxZu8Xid40A55PGE9mBEW-yuuJnsdnhAJJ-GLP93MdMr1qpEr_rtTkz9kxYHbZ8BPSefQUzerllqRn4oEierM7qbvtDgszyEInyFJFArqsGInzDuCAN8NKvgkZLouidllxOM8I0
   $ tar -zxvf pgilinux-2019-194-x86-64.tar.gz
   $ ./install

编译完成后，可以通过 ``~/pgi/lincense.info`` 查看lincense相关信息，将自动生成的 ``~/pgi/lincense.dat`` 替换rpm节点 ``/lustre/spack/etc/spack/licenses/pgi/license.dat`` 即可；
如果仍未生效可以运行 ``./install --env env/pi2-system.yaml --refresh`` 重新生成软件模块。
因社区版软件的license具有时效性，过期后可以通过两种方法更新：

1. 通过以上方法重新编译pgi获取新license。

2.  pgi本身提供了license生成和升级工具::

    $ cd ~/pgi/linux86-64-llvm/19.4/bin
    $ ./pgi_license_tool

此方法更新license需要pgi账户。

合并重复安装的软件包
--------------------

Spack上游软件包的variants发生变化后，Spack会认为当前安装的软件包不再满足spec要求，导致软件包被重复安装，以 intel-parallel-studio 为例，重复被安装多次会导致匹配问题。可调整环境变量中预期variants，使Spack优先匹配到已经安装的软件包。

检查已安装软件包使用的variants::

  $ spack find -v intel-parallel-studio@cluster.2019.5
  ==> 1 installed package
  -- linux-centos7-cascadelake / intel@19.0.5 ---------------------
  intel-parallel-studio@cluster.2019.5+advisor auto_dispatch=none ~clck+daal~gdb~ilp64+inspector+ipp+itac+mkl+mpi~newdtags+rpath+shared+tbb threads=openmp +vtune

测试Spack期望使用的variants::

  $ spack spec intel-parallel-studio@cluster.2019.5 %intel@19.0.5
  Input spec
  --------------------------------
  intel-parallel-studio@cluster.2019.5%intel@19.0.5
  
  Concretized
  --------------------------------
  intel-parallel-studio@cluster.2019.5%intel@19.0.5+advisor auto_dispatch=none +clck+daal~gdb~ilp64+inspector+ipp+itac+mkl+mpi~newdtags+rpath+shared+tbb threads=openmp +vtune arch=linux-centos7-skylake_avx512

对比发现Spack预期多装了一个 ``+clck`` variant，这个在已安装软件中是没有的。
编辑 ``env/pi2-system.yaml`` 去掉这个variant，或者显式地说明不需要 ``~clck`` 特性，然后重新测试是否会有重复安装的问题。
从登陆节点申请资源后在计算节点上编译，已经能自动识别到cascadelake架构，不需要用target指定。

  $ srun -p small -n 4 --pty /bin/bash  
  $ ./install --env env/pi2-system.yaml
  $ spack install intel-parallel-studio@cluster.2019.5 %intel@19.0.5
[+] /lustre/opt/cascadelake/linux-centos7-cascadelake/intel-19.0.5/intel-parallel-studio-cluster.2019.5-ju4d5rkrmkchhklxtevuqnnqtehx5nkq

处理Modules名字冲突的软件包
---------------------------

名字相同但variants不同的软件包，在Spack生成Modules时可能会导致名字冲突，可尝试通过如下三种方法解决。

- 方法一：把多个variants不同的同名同版本软件包归并到一个共同的超集软件上，在环境文件中指定安装该软件时使用超集的variants。
- 方法二：修改环境文件，只为满足特定条件(编译器、variant等)的软件包生成Modules。
- 方法三：手动移除重复软件包。这里以 ``python@3.7.4 %intel@19.0.5`` 重复软件包的处理过程为例，展示发现软件包、确认重复原因、删除重复包的过程。

发现重复软件包::

  $ spack find python %intel@19.0.5
  ==> 2 installed packages
  -- linux-centos7-cascadelake / intel@19.0.5 ---------------------
  python@3.7.4  python@3.7.4

确认重复原因，首先使用 ``spack find -v`` 检查是否是因为variant不同导致重复，对比以后发现这两个包的variants是一样的::

  $ spack find -v python@3.7.4 %intel@19.0.5
  ==> 2 installed packages
  -- linux-centos7-cascadelake / intel@19.0.5 ---------------------
  python@3.7.4+bz2+ctypes+dbm~debug+libxml2+lzma~nis~optimizations patches=ad05cbbb01a6a2a2496274098deea981d205f7c936acf56d976cdb540b87b843 +pic+pyexpat+pythoncmd+readline+shared+sqlite3+ssl~tix~tkinter~ucs4~uuid+zlib
  python@3.7.4+bz2+ctypes+dbm~debug+libxml2+lzma~nis~optimizations patches=ad05cbbb01a6a2a2496274098deea981d205f7c936acf56d976cdb540b87b843 +pic+pyexpat+pythoncmd+readline+shared+sqlite3+ssl~tix~tkinter~ucs4~uuid+zlib

Spack是根据软件包的依赖关系图来生成Hash，进而判断两个包是否“一样”。
进一步使用 ``spack find -dl`` 检查两个包编译时引入的依赖关系，发现是ncurses和openssl版本的微小变化导致的差异::

  $ spack find -dl python %intel@19.0.5                                                                      
  ==> 2 installed packages
  -- linux-centos7-cascadelake / intel@19.0.5 ---------------------
  x23u56z python@3.7.4
  47poaym     bzip2@1.0.8
  3bpkrs3     expat@2.2.9
  w6tur3a         libbsd@0.10.0
  lx2nlwu     gdbm@1.18.1
  rt4fzwj         readline@8.0                                                                                       be43mio             ncurses@6.1
  5u7xhrn     gettext@0.20.1                                                                                         ou6olia         libxml2@2.9.9
  plt3hnw             libiconv@1.16
  hqy4spw             xz@5.2.4
  p7mupkk             zlib@1.2.11
  df37h7j         tar@1.32
  ne3roxv     libffi@3.2.1
  6g23hf3     openssl@1.1.1d
  ww3j5gr     sqlite@3.30.1
  
  dcmb22g python@3.7.4
  47poaym     bzip2@1.0.8
  3bpkrs3     expat@2.2.9
  w6tur3a         libbsd@0.10.0
  t5craen     gdbm@1.18.1
  hmnszx2         readline@8.0
  bbpvmkf             ncurses@6.2
  ecvboc4     gettext@0.20.2
  plt3hnw         libiconv@1.16
  x47quyd         libxml2@2.9.10
  6p7xsge             xz@5.2.5
  p7mupkk             zlib@1.2.11
  df37h7j         tar@1.32
  jdzi5nj     libffi@3.3
  qc2p3sv     openssl@1.1.1g
  l3lqlab     sqlite@3.31.1

然后尝试手动删除其中一个软件包，使用 ``spack find -dl`` 查找到的Hash指定软件包，如果要删除的软件包没有其他软件包依赖，就键入 ``y`` 删除。

  $ spack uninstall /x23u56z
  ==> Will not uninstall python@3.7.4%intel@19.0.5/x23u56z
  The following packages depend on it:
      -- linux-centos7-cascadelake / intel@19.0.5 ---------------------
      2xfrn3q lammps@20190807%intel +asphere+body build_type=RelWithDebInfo +class2+colloid+compress+coreshell~cuda cuda_arch=none ~cuda_mps+dipole+exceptions~ffmpeg+granular+jpeg~kokkos+kspace+latte+lib+manybody+mc+meam+misc+molecule+mpi+mpiio~opencl+openmp+peri+png+poems+python+qeq+replica+rigid+shock+snap~spin+srd+user-atc+user-h5md+user-lb+user-misc+user-netcdf+user-omp+user-reaxc~voronoi
  
  ==> Error: There are still dependents.
    use `spack uninstall --dependents` to remove dependents too

删下面这一个：

  $ spack uninstall /dcmb22g
  ==> The following packages will be uninstalled:
  
      -- linux-centos7-cascadelake / intel@19.0.5 ---------------------
      dcmb22g python@3.7.4%intel +bz2+ctypes+dbm~debug+libxml2+lzma~nis~optimizations patches=ad05cbbb01a6a2a2496274098deea981d205f7c936acf56d976cdb540b87b843 +pic+pyexpat+pythoncmd+readline+shared+sqlite3+ssl~tix~tkinter~ucs4~uuid+zlib
  ==> Do you want to proceed? [y/N] y

最后重新生成Modules::

  $ spack module tcl refresh python@3.7.4 %intel@19.0.5
  ==> You are about to regenerate tcl module files for:
  
  -- linux-centos7-cascadelake / intel@19.0.5 ---------------------
  x23u56z python@3.7.4
  
  ==> Do you want to proceed? [y/n] y
  ==> Regenerating tcl module files

对 intel 模块的特殊处理
-----------------------

因Spack尚未修复的缺陷 https://github.com/spack/spack/issues/12628 ，intel系列产品(``intel-parallel-studio``, ``intel-mpi``, ``intel-mkl`` 等)的Environment Modules会错误地包含 ``/usr/bin`` 等系统路径，导致卸载模块后无法找到系统命令，需要手工修复::

  $ cd /lustre/share/spack/modules/cascadelake
  $ find . -type f ! -path './.git/*' -exec sed -i '/PYTHONHOME/d' {} \;
  $ find . -type f ! -path './.git/*' -exec sed -i '/PYTHONPATH/d' {} \;
  $ find . -type f ! -path './.git/*' -exec sed -i '/_CONDA_PYTHON/d' {} \;
  $ find . -type f ! -path './.git/*' -exec sed -i '/^append-path PATH "\/usr*/d' {} \;
  $ find . -type f ! -path './.git/*' -exec sed -i '/^append-path PATH "\/bin*/d' {} \;
  $ find . -type f ! -path './.git/*' -exec sed -i '/^append-path PATH "\/opt*/d' {} \;
  $ find . -type f ! -path './.git/*' -exec sed -i '/^append-path PATH "\/lustre\/home*/d' {} \;
  $ find . -type f ! -path './.git/*' -exec sed -i '/^append-path PATH "\/lustre\/opt*/d' {} \;

参考资料
========

* "Spack Modules" https://spack.readthedocs.io/en/latest/module_file_support.html
