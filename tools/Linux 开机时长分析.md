## 开机时长分析
systemd-analyze 是一个分析启动性能的工具，用于分析启动时服务时间消耗。默认显示启动是内核和用户空间的消耗时间：使用 systemd-analyze plot > boot.svg 生成一张启动详细信息矢量图，然后用图像浏览器或者网页浏览器打开查看 。
```
root@zyt:/home/zyt# systemd-analyze
Startup finished in 3.816s (kernel) + 3min 10.127s (userspace) = 3min 13.944s
graphical.target reached after 9.818s in userspace
```

（1）查看详细的每个服务消耗的启动时间
通过 systemd-analyze blame 命令查看详细的每个服务消耗的启动时间：
```
root@zyt:/home/zyt# systemd-analyze blame
          9.929s apt-daily.service
          2.056s dev-mapper-ubuntu\x2d\x2dvg\x2dubuntu\x2d\x2dlv.device
          2.004s systemd-networkd-wait-online.service
          1.921s snapd.service
          1.888s docker.service
          1.439s motd-news.service
          1.068s cloud-init-local.service
          1.056s containerd.service
```
（2）查看严重消耗时间的服务树状表
systemd-analyze critical-chain 以树状形式显示时间关键链。"@"后面的时刻表示该单元的启动时刻；"+"后面的时长表示该单元总计花了多长时间才完成启动。不过需要注意的是，这些信息也可能具有误导性，因为花费较长时间启动的单元， 有可能只是在等待另一个依赖单元完成启动。
```
root@zyt:/home/zyt# systemd-analyze critical-chain
The time after the unit is active or started is printed after the "@" character.
The time the unit takes to start is printed after the "+" character.

graphical.target @9.818s
└─multi-user.target @9.818s
  └─snap-core20-1518.mount @44.539s +20ms
    └─local-fs-pre.target @733ms
      └─keyboard-setup.service @367ms +365ms
        └─systemd-journald.socket @364ms
          └─system.slice @361ms
            └─-.slice @352ms
```
