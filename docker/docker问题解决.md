[TOC]

### 突然MySQL重启并且失败

- 使用docker logs mysql8查看报错日志

  ```
  mbind: Operation not permitted
  ```

- 尝试重启MySQL-

  ```
  Error response from daemon: Cannot restart container 1637c7a676b5: driver failed programming external connectivity on endpoint mysql8 (0b1d93ea6909579717226ec70a3e52471ea97a7a93672a14bccec9e47f75c136):  (iptables failed: iptables --wait -t nat -A DOCKER -p tcp -d 0/0 --dport 3306 -j DNAT --to-destination 172.18.0.2:3306 ! -i docker0: iptables: No chain/target/match by that name.
  ```

解决方法一

​	重启docker服务，但此方法在生产环境不适合使用，如果是测试环境可通过此方法解决

```shell
systemctl restart docker
```

解决方法二

[可能与防火墙有关，点击此链接查看](https://cloud.tencent.com/developer/article/1860370?from=article.detail.1494975)



