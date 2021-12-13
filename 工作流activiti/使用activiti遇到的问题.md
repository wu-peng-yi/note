[TOC]

### windows流程图正常，生产环境出现小框框

#### 原因

Linux环境缺少字体

#### 解决办法

下载字体放到Linux的JAVA_HOME/jre/lib/fonts/fallback/文件夹下并重启应用

```shell
## 查看JAVA_HOME
echo ${JAVA_HOME}

##进入目录
cd /usr/java/jdk1.8.0_161/jre/lib/fonts/fallback/

## 复制文件

##重启应用
```

