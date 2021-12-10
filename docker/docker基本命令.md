[TOC]

[]里面的内容表示可选的

- 获取容器/镜像的元数据。

  **docker inspect container_id/container_name  [ | grep  想查看的内容 -A 20]**

  * 可选 -A 20 表示从想查看内容向下显示20行

  ```shell
  docker inspect mysql8 | grep Mount -A 20
  ```

  ![image-20211210112458200](https://raw.githubusercontent.com/wu-peng-yi/images/main/image-20211210112458200.png)

- 容器与主机之间数据拷贝

  **docker cp src_path container_id:dest_path** 将主机数据拷贝到容器

  **docker cp container_id:src_path dest_path** 将容器数据拷贝到主机

  ```shell
  docker cp /root/test.txt 2c83628e2720:/root
  ```

  ![image-20211210113629604](https://raw.githubusercontent.com/wu-peng-yi/images/main/image-20211210113629604.png)

  

  

  







































