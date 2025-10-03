# ROS2 Docker Windows 部署完整教程

> 本教程基于真实部署经验，记录了从零开始搭建ROS2 Docker环境的完整过程，包含遇到的问题和解决方案。

## 📖 教程概述

本教程将指导你在Windows系统上使用Docker和WSL2部署ROS2 Humble开发环境。这是一个**真实案例教程**，包含了实际部署过程中遇到的所有问题和解决方法。

### 适用人群
- Windows用户想要学习ROS2
- 需要容器化ROS2开发环境的开发者
- 对Docker + ROS2组合感兴趣的技术人员

### 前置要求
- Windows 10/11 (版本2004或更高)
- 已安装Docker Desktop
- 已安装WSL2
- 管理员权限

---

## 🚀 第一部分：快速开始

如果你只是想快速验证ROS2是否能在你的系统上运行，按照以下步骤：

### 1.1 最简单的验证方法
```bash
# 启动一个临时ROS2容器
docker run -it --rm ros:humble bash

# 在容器内执行
source /opt/ros/humble/setup.bash
ros2
```

看到ROS2帮助信息输出，说明你的系统支持ROS2!

### 1.2 运行第一个ROS2程序
```bash
# 在同一个容器内继续执行
apt-get update
apt-get install -y ros-humble-demo-nodes-py

# 启动talker
ros2 run demo_nodes_py talker
```

看到持续的"Hello World"输出，恭喜你成功运行了第一个ROS2程序！

---

## 🏗️ 第二部分：完整部署过程

### 2.1 项目结构设计
我们设计的多容器架构如下：

```
ROS2/
├── Dockerfile              # ROS2镜像配置
├── docker-compose.yml      # 容器编排配置
├── README.md              # 项目文档
└── ros2_ws/              # ROS2工作空间
```

### 2.2 创建Dockerfile
```dockerfile
# ROS2 Humble Docker配置文件
FROM ros:humble

# 设置非root用户 (重要：使用宿主机UID/GID)
ARG USERNAME=ros
ARG USER_UID=197108        # 从宿主机获取
ARG USER_GID=197121        # 从宿主机获取

# 创建工作目录
WORKDIR /home/$USERNAME/ros2_ws

# 安装基础工具
RUN apt-get update && apt-get install -y \
    python3-pip \
    python3-vcstool \
    python3-colcon-common-extensions \
    build-essential \
    cmake \
    git \
    wget \
    curl \
    vim \
    htop \
    tree \
    && rm -rf /var/lib/apt/lists/*

# 安装Python工具
RUN pip3 install \
    argcomplete \
    flake8 \
    pytest \
    setuptools

# 关键：安装ROS2桌面工具和示例包
RUN apt-get update && apt-get install -y \
    ros-humble-desktop \
    ros-humble-demo-nodes-py \    # 重要：示例包
    && rm -rf /var/lib/apt/lists/*

# 创建用户和权限设置
RUN groupadd -g $USER_GID $USERNAME \
    && useradd -m -s /bin/bash -u $USER_UID -g $USER_GID $USERNAME \
    && echo "$USERNAME ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/90-user

# 设置环境变量
ENV ROS_DISTRO=humble
ENV ROS_VERSION=2
ENV ROS_PYTHON_VERSION=3

# 配置bash环境 (作为root用户操作)
RUN echo "source /opt/ros/humble/setup.bash" >> /home/$USERNAME/.bashrc
RUN echo "export colcon_ws=/home/$USERNAME/ros2_ws" >> /home/$USERNAME/.bashrc
RUN echo "cd \$colcon_ws" >> /home/$USERNAME/.bashrc

# 创建工作空间
RUN mkdir -p /home/$USERNAME/ros2_ws/src
RUN chown -R $USERNAME:$USER_GID /home/$USERNAME

# 切换到非root用户
USER $USERNAME

# 设置工作目录
WORKDIR /home/$USERNAME/ros2_ws

CMD ["/bin/bash"]
```

### 2.3 创建docker-compose.yml
```yaml
version: '3.8'

services:
  # ROS2主节点容器
  ros2-master:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: ros2_master
    environment:
      - DISPLAY=$DISPLAY
      - QT_X11_NO_MITSHM=1
    volumes:
      - /tmp/.X11-unix:/tmp/.X11-unix:rw
      - ./ros2_ws:/home/ros/ros2_ws:rw
      - /etc/passwd:/etc/passwd:ro
      - /etc/group:/etc/group:ro
    networks:
      - ros2_network
    stdin_open: true
    tty: true
    privileged: true
    ports:
      - "8080:8080"
      - "9090:9090"

  # ROS2工作节点容器
  ros2-node:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: ros2_node
    depends_on:
      - ros2-master
    environment:
      - DISPLAY=$DISPLAY
      - QT_X11_NO_MITSHM=1
      - ROS_MASTER_URI=http://ros2-master:11311
    volumes:
      - /tmp/.X11-unix:/tmp/.X11-unix:rw
      - ./ros2_ws:/home/ros/ros2_ws:rw
      - /etc/passwd:/etc/passwd:ro
      - /etc/group:/etc/group:ro
    networks:
      - ros2_network
    stdin_open: true
    tty: true

networks:
  ros2_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### 2.4 构建和启动
```bash
# 构建镜像
docker-compose build --no-cache

# 启动容器
docker-compose up -d

# 检查状态
docker-compose ps
```

---

## 🐛 第三部分：常见问题与解决方案

基于我们的实际部署经验，以下是可能遇到的问题和解决方法：

### 3.1 网络连接问题

**问题现象**：
```
dial tcp 162.125.80.6:443: connectex: A connection attempt failed
```

**解决方案**：
1. 启用VPN访问Docker Hub
2. 或配置Docker镜像加速器：
```json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
```

### 3.2 用户权限问题

**问题现象**：
```
groups: cannot find name for group ID 1000
I have no name!@containerid:~$
```

**根本原因**：容器内用户UID/GID与宿主不匹配

**解决方案**：
1. 查看宿主用户UID/GID：
```bash
# 在Windows PowerShell中
$id = [System.Security.Principal.WindowsIdentity]::GetCurrent()
$user = $id.Name
# 或者通过WSL查看
id
```

2. 修改Dockerfile中的UID/GID为宿主实际值：
```dockerfile
ARG USER_UID=197108  # 替换为你的实际UID
ARG USER_GID=197121  # 替换为你的实际GID
```

### 3.3 终端交互问题

**问题现象**：
```
the input device is not a TTY. If you are using mintty, try prefixing the command with 'winpty'
```

**解决方案**：
```bash
# 方法1：使用winpty
winpty docker exec -it ros2_master bash

# 方法2：去掉-it参数（无交互式终端）
docker exec ros2_master bash

# 方法3：使用命令直接执行
docker exec ros2_master bash -c "source /opt/ros/humble/setup.bash && ros2"
```

### 3.4 ROS2包依赖问题

**问题现象**：
```
E: Unable to locate package ros-humble-viz-tools
Package 'demo_nodes_py' not found
```

**解决方案**：
1. 检查包是否存在：
```bash
apt-cache search ros-humble | grep demo
```

2. 只安装存在的包：
```dockerfile
# 正确的包列表
RUN apt-get update && apt-get install -y \
    ros-humble-desktop \
    ros-humble-demo-nodes-py \    # 这个包存在
    # ros-humble-viz-tools \       # 这个包不存在，不要安装
    && rm -rf /var/lib/apt/lists/*
```

### 3.5 环境变量问题

**问题现象**：
```
ros2: command not found
ros2: error: unrecognized arguments: --version
```

**解决方案**：
```bash
# 每次进入容器都要加载环境
source /opt/ros/humble/setup.bash

# 注意：ROS2没有--version参数，直接运行ros2查看帮助
ros2
```

---

## 🧪 第四部分：功能验证

### 4.1 基础功能测试
```bash
# 1. 验证容器状态
docker-compose ps

# 2. 测试ROS2基础环境
docker exec ros2_master bash -c "source /opt/ros/humble/setup.bash && ros2"

# 3. 列出可用包
docker exec ros2_master bash -c "source /opt/ros/humble/setup.bash && ros2 pkg list | head -5"

# 4. 测试节点和话题命令
docker exec ros2_master bash -c "source /opt/ros/humble/setup.bash && ros2 topic list"
docker exec ros2_master bash -c "source /opt/ros/humble/setup.bash && ros2 node list"
```

### 4.2 完整通信测试
```bash
# 终端1：启动talker
docker exec ros2_master bash -c "source /opt/ros/humble/setup.bash && ros2 run demo_nodes_py talker"

# 终端2：启动listener (新PowerShell窗口)
docker exec ros2_node bash -c "source /opt/ros/humble/setup.bash && ros2 run demo_nodes_py listener"
```

看到talker发送消息，listener接收消息，说明通信成功！

---

## 📚 第五部分：进阶主题

### 5.1 单容器简化方案

如果多容器架构太复杂，可以使用单容器方案：

```bash
# 创建简化docker-compose.yml
version: '3.8'
services:
  ros2-simple:
    build: .
    container_name: ros2_simple
    volumes:
      - ./ros2_ws:/home/ros/ros2_ws:rw
    stdin_open: true
    tty: true
    ports:
      - "8080:8080"
```

### 5.2 GUI应用支持

要运行RViz2等GUI应用，需要配置X11转发：

1. 安装VcXsrv (Windows X11服务器)
2. 启动XLaunch，选择"Multiple windows"
3. 勾选"Disable access control"
4. 在容器中测试：
```bash
docker exec -it ros2_master bash
source /opt/ros/humble/setup.bash
xeyes  # 测试X11转发
rviz2   # 启动RViz2
```

### 5.3 开发工作流

```bash
# 1. 创建ROS2包
docker exec ros2_master bash -c "cd ~/ros2_ws/src && source /opt/ros/humble/setup.bash && ros2 pkg create --build-type ament_python my_package"

# 2. 编辑代码 (在Windows的ros2_ws目录)

# 3. 编译工作空间
docker exec ros2_master bash -c "cd ~/ros2_ws && colcon build"

# 4. 源工作空间并运行
docker exec ros2_master bash -c "source ~/ros2_ws/install/setup.bash && ros2 run my_package my_node"
```

---

## 🎯 第六部分：项目成果总结

### 6.1 我们学到了什么

1. **复杂性的代价**：过度设计会导致更多问题
2. **渐进式开发**：应该从简单开始，逐步增加复杂度
3. **实际问题驱动**：真实环境中的问题与理论假设差异很大
4. **文档的重要性**：记录问题和解决方案对后续维护很关键

### 6.2 技术要点

- ✅ Docker容器化ROS2的完整流程
- ✅ Windows + WSL2 + Docker的集成使用
- ✅ 用户权限和UID/GID映射问题解决
- ✅ ROS2包管理和环境变量配置
- ✅ 多容器架构的设计与实现

### 6.3 最佳实践建议

1. **从简单开始**：先用单容器验证基本功能
2. **分步测试**：每完成一步都要验证功能
3. **记录问题**：详细记录遇到的问题和解决方法
4. **环境隔离**：使用Docker确保环境的一致性
5. **权限管理**：正确处理Windows和容器间的用户映射

---

## 🛠️ 附录：常用命令速查

### 容器管理命令
```bash
# 查看容器状态
docker-compose ps

# 启动/停止容器
docker-compose up -d
docker-compose down

# 重新构建
docker-compose build --no-cache

# 查看日志
docker-compose logs ros2_master
```

### ROS2常用命令
```bash
# 进入容器并加载环境
docker exec ros2_master bash -c "source /opt/ros/humble/setup.bash && ros2"

# 包管理
ros2 pkg list
ros2 pkg create --build-type ament_python my_package

# 节点和话题
ros2 node list
ros2 topic list
ros2 topic echo /chatter

# 运行节点
ros2 run demo_nodes_py talker
ros2 run demo_nodes_py listener
```

### 故障排查命令
```bash
# 检查Docker状态
docker version
docker info

# 检查网络连接
docker network ls
docker network inspect ros2_ros2_network

# 清理资源
docker system prune
docker volume prune
```

---

## 📖 结语

本教程记录了一次真实的ROS2 Docker部署过程，包含了成功经验和失败教训。通过这个项目，我们不仅成功部署了ROS2环境，更重要的是学会了如何系统性地解决问题。

**关键收获**：
- 技术只是工具，解决问题的思路更重要
- 复杂性是项目的敌人，简洁是美德
- 实战经验比理论知识更宝贵

希望这个教程对你在ROS2学习和部署过程中有所帮助！

---

**文档版本**: 1.0
**最后更新**: 2025-10-03
**作者**: Claude + 实际项目经验
**贡献**: 欢迎提交Issue和Pull Request