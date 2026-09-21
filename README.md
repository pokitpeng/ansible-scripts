# 常用 Ansible 脚本

从 `self-built-cluster/qi-hao` 中整理的通用主机管理 playbook。仓库仅包含示例变量，不包含原集群的真实 inventory、密码或 Kubernetes 测试资源。

## 开始使用

1. 安装 Ansible，并复制 `hosts.ini.example` 为 `hosts.ini`，填写自己的主机地址和登录方式。`hosts.ini` 已被 Git 忽略。
2. 编辑目标 playbook 对应的 `*-vars.yaml`，核对主机组、文件路径和参数。修改型任务还需将对应的 `*_enabled` 设为 `true`；`00-ping.yaml` 无需启用开关。
3. 先检查语法和匹配的主机，再运行。例如：

   ```bash
   ansible-playbook -i hosts.ini --syntax-check 60-sysctl.yaml
   ansible-playbook -i hosts.ini --list-hosts 60-sysctl.yaml
   ansible-playbook -i hosts.ini 60-sysctl.yaml --limit node1 --check --diff
   ansible-playbook -i hosts.ini 60-sysctl.yaml --limit node1
   ```

`--check` 不能可靠预演 shell 命令、磁盘格式化或所有系统操作；这些任务要根据变量和 `--list-hosts` 结果人工确认。带 `localhost` 的预检 play 在使用 `--limit` 时可能不执行，所以每个目标 play 也有启用条件校验。

## 新机器初始化顺序

先完成 inventory 配置，再按下表逐台执行。主要步骤按 10 递增，并在编号空隙中加入独立步骤。标为“按需”的步骤没有必要为每台机器都运行；每一步都先核对对应的 `*-vars.yaml` 和匹配的主机。

| 编号 | 脚本 | 用途与执行条件 |
| --- | --- | --- |
| 00 | `00-ping.yaml` | 通过 `00-ping-vars.yaml` 设置检查范围；inventory 启用 `ansible_become` 时也会检查提权。例：`ansible-playbook -i hosts.ini 00-ping.yaml --limit node1`。 |
| 10（按需） | `10-add-local-ssh-key.yaml` | 把控制机本机公钥（默认 `~/.ssh/id_ed25519.pub`）写入目标登录用户 `authorized_keys`；可选同时写入 root。 |
| 20（按需） | `20-set-root-password.yaml` | 需要设置 root 密码或调整 root SSH 登录策略时执行。若已能用普通用户加 sudo 管理主机，可跳过。默认不允许 root 密码登录。 |
| 30 | `30-set-hostname.yaml` | 根据 inventory 名称或映射设置系统 hostname；先确认命名。 |
| 40（按需） | `40-setup-root-ssh-key.yaml` | 需要从一台源机以 root 免密访问其他节点时执行；`--limit` 必须同时包含源机和目标机。 |
| 50（按需） | `50-configure-dns.yaml` | 需要修改 Ubuntu netplan DNS 时执行，尤其应在下载 Docker 前确认解析可用。先确认 `configure_dns_netplan_file`（cloud-init 常见为 `50-cloud-init.yaml`）；远端需有 `python3-yaml`，验证需有 `dig`。改动网络配置可能中断当前连接。 |
| 60（按需） | `60-sysctl.yaml` | 写入需要的内核参数；只配置本机实际需要的参数。 |
| 70（按需） | `70-disable-auto-updates.yaml` | 仅在已有其他更新维护方式时关闭 APT、固件刷新、系统维护及更新通知单元；不存在的单元会跳过。 |
| 80（按需） | `80-init-nvme-disks.yaml` | 确认盘符和数据后初始化、挂载空 NVMe 盘；需要 `ansible.posix` 集合。若 Docker 的 `data-root` 在此盘上，必须先完成挂载。 |
| 90（按需） | `90-install-docker.yaml` | 安装静态 Docker 26.1.4，配置 daemon 和 systemd 服务；确认下载源与数据目录。 |
| 91（按需） | `91-install-nvidia-container-toolkit.yaml` | Ubuntu/Debian 上从 Gitee 安装 NVIDIA Container Toolkit 1.20.1；确认对应架构压缩包已同步。 |
| 92（按需） | `92-configure-nvidia-runtime.yaml` | Docker 和 NVIDIA Container Toolkit 就绪后，注册 NVIDIA runtime 并按需设为默认。 |
| 95（按需） | `95-install-buildx.yaml` | Docker CLI 就绪后，单独安装或更新 root 的 Buildx 0.37.1 插件。 |
| 97（按需） | `97-install-docker-compose.yaml` | Docker CLI 就绪后，从 Gitee 单独安装或更新 Docker Compose 5.5.1 独立命令。 |

`99-run-shell.yaml` 是临时批量命令工具，不属于固定初始化步骤；应在明确命令和目标范围后单独使用。除“磁盘挂载先于使用该目录的 Docker”外，按需步骤可根据机器情况调整。

安装额外集合：

```bash
ansible-galaxy collection install ansible.posix
```

密码建议放在 Ansible Vault 加密的变量文件中，并通过 `-e @/path/to/secret-vars.yaml --ask-vault-pass` 传入；不要把密码写进命令行、`hosts.ini` 或提交到仓库。`20-set-root-password-vars.yaml` 的密码默认为空，所有修改型 playbook 默认关闭。

## Docker 与 Buildx

Docker 和 Buildx 分别由两个 playbook 管理。编辑 `90-install-docker-vars.yaml`，确认 `docker_hosts`、`docker_data_root` 和镜像地址，再设 `docker_install_enabled: true`。静态 Docker 默认使用清华镜像；也可将 `docker_source_url` 改为 `https://mirrors.aliyun.com`。

GPU 主机可先编辑 `91-install-nvidia-container-toolkit-vars.yaml`，设 `nvidia_toolkit_enabled: true`，再运行 `91-install-nvidia-container-toolkit.yaml`。脚本从 Gitee 下载官方 deb 压缩包并校验 SHA256，一次安装 `libnvidia-container1`、`libnvidia-container-tools`、`nvidia-container-toolkit-base` 和 `nvidia-container-toolkit`。四个包版本一致时跳过下载和安装；自动降级、删除已有包的操作会被 APT 拒绝。系统依赖通过目标机已有 APT 源获取，因此这不是完全离线安装，需保证 APT 源与索引可用。宿主机 NVIDIA 驱动需另外安装。

运行 `91` 前，需将 `NVIDIA/nvidia-container-toolkit` 的 v1.20.1 Release 中的 `nvidia-container-toolkit_1.20.1_deb_amd64.tar.gz` 同步到 `banbaolatiao/nvidia-container-toolkit`（arm64 对应 `deb_arm64.tar.gz`）。缺少文件时下载直接报错，不切换源。更换版本时同时更新 vars 中的版本、deb 修订号和 SHA256。`--check --diff` 仅读取已安装版本并报告是否需要安装，不下载或写文件，也不验证源是否可用或模拟依赖解析。

Docker 和 Toolkit 就绪后，编辑 `92-configure-nvidia-runtime-vars.yaml` 并运行 `92-configure-nvidia-runtime.yaml`。脚本使用 NVIDIA 官方的 `nvidia-ctk runtime configure --runtime=docker` 流程，只有 daemon 配置发生变化时才重启 Docker；是否把 `nvidia` 设为默认由 `nvidia_runtime_set_as_default` 控制。`90` 会合并其管理的 daemon 配置键并保留 NVIDIA runtime 等其他已有键。

Docker CLI 就绪后，编辑 `95-install-buildx-vars.yaml`，设 `docker_buildx_enabled: true`，再单独运行 `95-install-buildx.yaml`。Buildx 默认从 Gitee 下载 v0.37.1 并校验官方 SHA256；脚本按目标主机架构选择文件。Gitee 缺少对应版本或架构文件时下载会报错，不会切换到其他源。更换 Buildx 版本时，需同时更新 `docker_buildx_sha256`。

Docker Compose 也独立管理。编辑 `97-install-docker-compose-vars.yaml`，设 `docker_compose_enabled: true`，再运行 `97-install-docker-compose.yaml`。默认从 Gitee 下载 v5.5.1 到 `/usr/local/bin/docker-compose` 并校验官方 SHA256。Gitee 缺少对应版本或架构文件时下载会报错，不会切换到其他源。

```bash
ansible-playbook -i hosts.ini --syntax-check 90-install-docker.yaml
ansible-playbook -i hosts.ini --list-hosts 90-install-docker.yaml
ansible-playbook -i hosts.ini 90-install-docker.yaml --limit node1
ansible-playbook -i hosts.ini 91-install-nvidia-container-toolkit.yaml --limit node1 --check --diff
ansible-playbook -i hosts.ini 91-install-nvidia-container-toolkit.yaml --limit node1
ansible-playbook -i hosts.ini 92-configure-nvidia-runtime.yaml --limit node1 --check --diff
ansible-playbook -i hosts.ini 92-configure-nvidia-runtime.yaml --limit node1
ansible-playbook -i hosts.ini --list-hosts 95-install-buildx.yaml
ansible-playbook -i hosts.ini 95-install-buildx.yaml --limit node1
ansible-playbook -i hosts.ini --list-hosts 97-install-docker-compose.yaml
ansible-playbook -i hosts.ini 97-install-docker-compose.yaml --limit node1
```

如果目标机已有 `docker` 命令，`90` 沿用现有 Docker 安装；否则下载静态包到 `/opt/docker`，创建 `/usr/local/bin/docker` 链接和 systemd 服务。`90` 会管理 `/etc/docker/daemon.json`，改动时重启 Docker；写入该文件会替换现有内容，Ansible 会备份旧文件。daemon 的 `experimental`、`debug` 和 BuildKit 设置仍在 `90-install-docker-vars.yaml` 中。`95` 只安装 root 的 Buildx 插件并更新 root 的 Docker CLI 配置，`97` 只安装独立的 `docker-compose` 命令；两者都不重启 Docker。
