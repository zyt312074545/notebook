1. 编译安装 containerd
编译文件包括：
- bin/ctr
- bin/containerd
- bin/containerd-stress
- bin/containerd-shim
- bin/containerd-shim-runc-v1
- bin/containerd-shim-runc-v2
安装目录：/usr/loca/bin
2. 编译安装 runc
编译文件包括：runc
安装目录：/usr/local/sbin
3. 编译安装 crictl
编译文件包括：
- build/bin/critest
- build/bin/crictl
安装目录：/usr/loca/bin
4. 配置 containerd service
安装目录：/etc/systemd/system
5. 配置 containerd toml 配置
containerd config default > /etc/containerd/config.toml
6. systemctl 加载 containerd 并启动
7. 配置 crictl 连接 containerd
安装目录： crictl.yaml
8. 配置镜像源（可选）
```toml
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://hub-mirror.c.163.com"]
```
9. 配置代理（可选）
```
Environment="HTTP_PROXY=http://127.0.0.1:7890/"
Environment="HTTPS_PROXY=http://127.0.0.1:7890/"
Environment="NO_PROXY=10.96.0.0/16,127.0.0.1,192.168.0.0/16,localhost"
```