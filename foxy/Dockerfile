# syntax = docker/dockerfile:1.4
FROM ubuntu:focal-20230412 AS builder

ARG TARGETPLATFORM
ARG DEBIAN_FRONTEND=noninteractive
LABEL maintainer="Andrew<andrew.diyi@outlook.com>"

SHELL ["/bin/bash", "-c"]

# 第一阶段：基础环境构建
RUN --mount=type=cache,target=/var/cache/apt \
  cp /etc/apt/sources.list /etc/apt/sources.list.bak && \
  sed -i "s@http://.*ports.ubuntu.com@http://mirrors.sustech.edu.cn@g" /etc/apt/sources.list && \
  apt-get update -q && \
  apt-get upgrade -y && \
  apt-get install -y apt-utils && \
  apt-get install -y \
  software-properties-common \
  apt-transport-https \
  ca-certificates \
  gnupg2 && \
  rm -rf /var/lib/apt/lists/*

# 第二阶段：桌面环境及核心组件安装
FROM builder AS core

RUN --mount=type=cache,target=/var/cache/apt \
  apt-get update -q && \
  apt-get install -y \
  cmake-curses-gui \
  ubuntu-mate-desktop \
  tigervnc-standalone-server \
  tigervnc-common \
  supervisor \
  wget \
  curl \
  gosu \
  git \
  sudo \
  python3-pip \
  tini \
  build-essential \
  vim \
  libglfw3-dev \
  lsb-release \
  locales \
  bash-completion \
  tzdata \
  terminator && \
  apt-get autoremove -y && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*

# 第三阶段：应用层安装
FROM core AS apps

# 配置 Python 环境
RUN --mount=type=cache,target=/root/.cache/pip \
  pip install --upgrade pip --index-url https://mirrors.sustech.edu.cn/pypi/web/simple && \
  pip config set global.index-url https://mirrors.sustech.edu.cn/pypi/web/simple && \
  pip install pyyaml typing-extensions git+https://github.com/novnc/websockify.git@v0.10.0

# 安装 noVNC
RUN git clone https://github.com/AtsushiSaito/noVNC.git -b add_clipboard_support /usr/lib/novnc && \
  ln -s /usr/lib/novnc/vnc.html /usr/lib/novnc/index.html && \
  sed -i "s/UI.initSetting('resize', 'off');/UI.initSetting('resize', 'remote');/g" /usr/lib/novnc/app/ui.js

# 安装浏览器及开发工具
RUN --mount=type=cache,target=/var/cache/apt \
  add-apt-repository ppa:mozillateam/ppa -y && \
  echo 'Package: *' > /etc/apt/preferences.d/mozilla-firefox && \
  echo 'Pin: release o=LP-PPA-mozillateam' >> /etc/apt/preferences.d/mozilla-firefox && \
  echo 'Pin-Priority: 1001' >> /etc/apt/preferences.d/mozilla-firefox && \
  wget -qO - https://gitlab.com/paulcarroty/vscodium-deb-rpm-repo/raw/master/pub.gpg | gpg --dearmor | tee /usr/share/keyrings/vscodium-archive-keyring.gpg >/dev/null && \
  echo 'deb [ signed-by=/usr/share/keyrings/vscodium-archive-keyring.gpg ] https://paulcarroty.gitlab.io/vscodium-deb-rpm-repo/debs vscodium main' > /etc/apt/sources.list.d/vscodium.list && \
  apt-get update -q && \
  apt-get install -y \
  firefox \
  codium && \
  apt-get autoremove -y && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*

# 第四阶段：ROS 安装
FROM apps AS ros

ENV ROS_DISTRO=foxy
ARG INSTALL_PACKAGE=desktop

RUN --mount=type=cache,target=/var/cache/apt \
  curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg && \
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] https://mirrors.sustech.edu.cn/ros2/ubuntu $(lsb_release -cs) main" > /etc/apt/sources.list.d/ros2.list && \
  apt-get update -q && \
  apt-get install -y \
  ros-${ROS_DISTRO}-${INSTALL_PACKAGE} \
  python3-argcomplete \
  python3-colcon-common-extensions \
  python3-rosdep \
  python3-vcstool \
  ros-${ROS_DISTRO}-gazebo-ros-pkgs \
  ros-${ROS_DISTRO}-ros-ign-gazebo && \
  rosdep init && \
  rosdep update && \
  apt-get autoremove -y && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*

# 第五阶段：最终配置
FROM ros

# 系统配置
RUN sed -i 's/Prompt=.*/Prompt=never/' /etc/update-manager/release-upgrades && \
  sed -i 's/enabled=1/enabled=0/g' /etc/default/apport && \
  rm /etc/apt/apt.conf.d/docker-clean

# 用户管理
ENV USER=gac
ENV PASSWORD=123

COPY ./entrypoint.sh /
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/bin/bash", "-c", "/entrypoint.sh"]