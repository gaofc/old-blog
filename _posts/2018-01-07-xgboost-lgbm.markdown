---
title: Windows下XGBoost和LightGBM环境配置
layout: post
guid: urn:uuid:f2d36d25-c14c-462f-bdbf-e6b676eeead6
tags:
  - xgboost
  - lightgbm
  - windows
  - 环境配置
img: 
---

### XGBoost和LightGBM简介

**XGBoost**是大规模并行Boosted Tree的工具，是一款经过优化的分布式梯度提升（Gradient Boosting）库，具有高效，灵活和高可移植性的特点。XGBoost基于梯度提升框架，实现了并行方式的决策树提升(Tree Boosting)，从而能够快速准确地解决各种数据科学问题。
在数据科学方面，有大量kaggle选手选用它进行数据挖掘比赛，其中包括两个以上kaggle比赛的夺冠方案。在工业界规模方面，XGBoos的分布式版本有广泛的可移植性，支持在YARN，MPI，Sungrid，Engine等各个平台上面运行，并且保留了单机并行版本的各种优化，使得它可以很好地解决于工业界规模的问题。

XGBoos的出现，让数据民工们告别了传统的机器学习算法们：RF、GBM、SVM、LASSO...之后微软推出了一个新的boosting框架，想要挑战xgboost的江湖地位，即**LightGBM（Light Gradient Boosting Machine）**。LightGBM同样是一款基于决策树算法的分布式梯度提升框架。具体性能对比各位可以自己测试一下。

### XGBoost环境配置

1. 下载源码
XGBoost源码地址在[GitHub](https://github.com/dmlc/xgboost/)上，也可以直接通过下面的地址来下载源码[https://github.com/dmlc/xgboost/archive/master.zip](https://github.com/dmlc/xgboost/archive/master.zip)

2. 解压文件
将文件解压到本地的一个地址，进入文件夹```xgboost-master/python-package```

3. 下载windows下的编译好的xgboost库文件
*很多参考资料上是通过源码编译dll文件，这是python安装需要依赖的一个文件。但是按照教程会出现各种各样的错误。免去这一步会方便很多。*
网上有直接编译好的dll文件，可以直接下载，所以就免去了编译的过程，方便很多。[http://ssl.picnet.com.au/xgboost/](http://ssl.picnet.com.au/xgboost/)这个页面会展示每日编译好的dll文件，可以直接下载最新的版本。如果你有NVIDIA的GPU，可以选择```GPU enabled```版本，如果没有的话，就选择```Not GPU-enabled```版本的。
之后在本机进入```xgboost-master\python-package```文件夹，将下载好的dll文件放入这个文件夹中。

4. 安装xgboost
通过cmd命令行进入```xgboost-master\python-package```文件夹，执行命令

		python setup.py install
这里有个坑需要注意，如果你使用的是python3的版本运行代码，最好使用python3的版本来安装，否则中途可能会出错。找到python3安装的路径，之后还是进入到```xgboost-master\python-package```文件夹，执行命令

		"C:\your python3 path\python.exe" setup.py install

5. IDE安装
本人使用的是PyCharm的IDE，找到```File->Settings->Projest:XXX->Projest Interpreter```，点击+，搜索```xgboost```，安装，之后再代码中```import xgboost```就可以使用了

![pycharm](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/2018010701.png)


### LightGBM环境配置

理论上，和xgboost安装方式是一样的，但是在网上没有找到dll文件，所以只能手动使用VS来编译了。如果你自己找到的话，可以按照之前的方法来安装。

1. 安装Visual Studio
需要使用VS2013（或者更高的版本）。现在Visual Studio出了Community版本面向个人和学生，是免费的。在[官方网站](https://www.visualstudio.com/zh-hans/downloads/)上可以下载Community 2017版本。之后安装就可以了。

2. 下载LightGBM源码
源码地址在[https://github.com/Microsoft/LightGBM](https://github.com/Microsoft/LightGBM)，可以使用git命令或者直接下载zip文件。

3. 解压文件
将文件解压到本地的一个地址，进入文件夹```LightGBM-master\windows```

4. 编译dll
使用VS打开```LightGBM.sln```文件，解决方案选择```DLL```，版本选x64，用快捷键```Ctrl+Shift+B```，生成解决方案。之后编译好的dll文件会在```windows\x64\DLL```文件夹里。
![VS](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/2018010702.png)

5. 编译exe
之后解决方案再选择```Release```，生成解决方案。exe文件会出现在```windows\x64\Release```文件夹中。如果发现了exe文件，说明安装成功了。

6. 安装python包
通过cmd命令行进入```python-package```文件夹，执行命令
		
		python setup.py install

5. IDE安装
找到```File->Settings->Projest:XXX->Projest Interpreter```，点击+，依次搜索```setuptools, wheel, numpy, scipy, scikit-learn, lightgbm```点击安装。之后再代码中```import lightgbm```就可以使用了


### 总结
环境配置过程是一个很头疼的事情，网上参考资料参差不齐，按照一个教程去执行，总是会出问题，把折腾的过程总结起来，供大家参考。如果按照作者的教程安装成功，恭喜你！如果没有安装成功，也是正常的，毕竟折腾环境是一个非常耗时的过程。如果大家在安装过程中出现问题，也欢迎和作者交流。另，机器学习方面的知识也欢迎一起交流。最后祝大家**Have fun in ML.**



