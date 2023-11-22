# GoosefsX 接入 ThinRuntime 的简单示例

## 前期准备
GoosefsX 接入 ThinRuntime 需要构造 Mount 客户端，直接命令就是 `mount --bind <from> <to>`

## 准备 GoosefsX Mount 客户端镜像
1. 挂载参数解析脚本

在 Mount 容器内需要提取相关的 **ThinRuntimeProfile、Dataset、ThinRuntime**资源中对远程文件系统的配置信息，相关信息以 JSON 字符串的方式保存到 FUSE 容器的 **/etc/fluid/config.json** 文件内。


```python
# fluid_config_init.py
import json

rawStr = ""
with open("/etc/fluid/config.json", "r") as f:
    rawStr = f.readlines()

rawStr = rawStr[0]

script = """
#!/bin/sh
set -ex

MNT_FROM="/goosefsx/${GOOSEFSX_ID}-proxy/${mountPoint}"
MNT_TO=$targetPath

trap "umount ${MNT_TO}" SIGTERM
mkdir -p ${MNT_TO}
mount --bind ${MNT_FROM} ${MNT_TO}
sleep inf
"""

obj = json.loads(rawStr)

with open("mount-nfs.sh", "w") as f:
    f.write("mountPoint=\"%s\"\n" % obj['mounts'][0]['mountPoint'])
    f.write("targetPath=\"%s\"\n" % obj['targetPath'])
    f.write(script)

```
该 Python 脚本，将参数提取后以变量的方式注入 shell 脚本。

2. 挂载远程文件系统脚本

在将参数解析并注入shell脚本后，生成的脚本如下
```shell
mountPoint="/goosefsx/<GOOSEFSX_ID>-proxy/<cos>"
targetPath="/runtime-mnt/thin/default/my-storage/thin-fuse"

#!/bin/sh
set -ex
MNT_FROM=$mountPoint
MNT_TO=$targetPath

trap "umount ${MNT_TO}" SIGTERM
mkdir -p ${MNT_TO}
mount --bind ${MNT_FROM} ${MNT_TO}
sleep inf
```
该 shell 脚本创建挂载的文件夹并将远程文件系统挂载到目标位置（targetPath）。**由于 mount 命令会立即返回，为了保持该进程的持续运行（防止Mount pod 反复重启），需要 sleep inf 来保持进程的存在。同时为了在 Mount pod 被删除前，将挂载的远程存储系统卸载，需要捕捉 SIGTERM 信号执行卸载命令**。

3. 创建 Mount 客户端镜像


将参数解析脚本、挂载脚本和相关的库打包入镜像。

```dockerfile
FROM ubuntu:jammy
RUN apt update && \
    apt install --yes libnfs13 libfuse2 fuse python3 bash && \
    apt clean autoclean && \
    apt autoremove --yes && \
    rm -rf /var/lib/{apt,dpkg,cache,log}/
ADD ./fluid_config_init.py /
ADD ./entrypoint.sh /usr/local/bin
CMD ["/usr/local/bin/entrypoint.sh"]
```
用户将上述 Dockerfile 、参数解析脚本 fluid_config_init.py和启动脚本 entrypoint.sh 复制到项目下父目录下**。
```shell
$ ls                                      
Dockerfile  entrypoint.sh  fluid_config_init.py
```

## 使用示例