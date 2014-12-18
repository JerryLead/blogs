# Petuum本地编译运行


## 编译Pettum
按照Pettum的安装文档[Installation](https://github.com/petuum/public/wiki/Installation)编译。该编译流程会先下载并编译第三方库（如boost，gflags，leveldb等等），然后会编译petuum自身（见petuum/Makefile）。

## 编译apps
上面的步骤只是编译Petuum计算框架，每个app需要单独编译。比如要编译Matrixfact，只需要进入apps/matrixfact，执行`make`就行了。具体参见[ML App: Matrix Factorization](https://github.com/petuum/public/wiki/ML-App:-Matrix-Factorization)。其他的apps（比如LDA，DNN类似）。

## 本地运行app测试
按照Pettum的[wiki](https://github.com/petuum/public/wiki)运行就好了。

## 在Eclipse里面编译petuum
想要深入理解代码，肯定要导入IDE中debug了。

1. 导入project

    下载Linux版本的Eclipse CDT，将编译好的petuum整个文件夹拷贝到workspace下面，然后删去third_party下面的src文件夹（整个文件夹存放了第三方库的源码，可以不要）。最后，将workspace/petuum import到Eclipse里面，选择`File->New project->C/C++->Makefile Project with Existing Code`，设置Toolchain for Indexer Settings为Linux GCC。
 
2. 编译Petuum

    导入后，直接`Project->Build project`就可以编译petuum了，以后可以修改petuum的源代码，然后直接build就行了。
    
3. 编译apps
    
    Eclipse不能自动识别子项目的Makefile，因此`Project->Build project`只能编译petuum自身不能编译apps。解决方法是手动添加apps的编译项。具体方法是在`Properties->C/C++ Build->Manage configurations->New`添加一个编译项，比如name设为matrixfact，确定后设置Build directory为`${workspace_loc:/petuum-0.93/apps/matrixfact}`，Refresh Policy里面的Resources设置为`${workspace_loc:/petuum-0.93/apps/matrixfact}`，最后设置matrixfact为active后，就可以编译matrixfact子项目了。

## 在Eclipse里面运行petuum

Petuum使用脚本运行方式来执行app，比如需要在某个node上执行

```
scripts/run_matrixfact.sh sampledata/9x9_3blocks 3 100 mf_output scripts/localserver 4 "--staleness 5 --lambda 0.1"
```
来运行matrixfact，那么如何在Eclipse里达到同样的效果？

答案是在`Run->External Tools->External Tools Configurations`里面的Program里面添加一个运行项，比如命名为run_matrixfact。然后设置Location为`${workspace_loc:/petuum-0.93/apps/matrixfact/scripts/run_matrixfact.sh}`，Workding Directory为`${workspace_loc:/petuum-0.93/apps/matrixfact}`， Arugments为`sampledata/9x9_3blocks 3 100 mf_output scripts/localserver 2`。然后run就相当于在terminal里面执行该脚本了。但遗憾的是没有debug as external tools。想要debug，目前可以用下面的方法解决。

## 在Eclipse里面debug app

首先要new一个Debug configuration，名字可以是app的名字，比如matrixfact。C/C++ Application是app的路径，比如`apps/matrixfact/bin/matrixfact`。Build Configuration选择之前添加的`matrixfact`。Program arguments填入

```
--hostfile scripts/localserver --datafile sampledata/9x9_3blocks --output_prefix mf_output --K 3 --num_iterations 100 --num_worker_threads 2 --num_clients 1 --client_id 0
```

Working directionary输入`${workspace_loc:petuum-0.93/apps/matrixfact}`。最后设置断点，开始debug。

> 需要注意的是以这种方式进行debug仅仅是在debug众多client中的一个。
    




