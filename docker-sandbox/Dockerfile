# ============================================================================
# OpenClaw 沙箱工作镜像
# 支持架构：linux/amd64, linux/arm64
# 多架构构建：
#   docker buildx build --platform linux/amd64,linux/arm64 \
#     -t aosolao/openclaw-sandbox:py313-node26-trixie --push .
# 单架构构建：
#   docker build -t aosolao/openclaw-sandbox:py313-node26-trixie .
# 基础镜像：Python 3.13 + Node 26，Debian 13 (trixie)
# 说明：OpenClaw 默认沙箱镜像为 debian:bookworm-slim + python3，
#       本镜像相比默认镜像额外预装 Node 26、金融数据/爬虫/文档处理等
#       工具链，以及 Xvfb 虚拟显示器和反检测字体，作为功能增强的
#       自定义沙箱镜像使用。trixie 与 OpenClaw 沙箱运行时兼容
#       （仅需 bash + python3 + 标准工具）。
# 软件源：apt / pip / npm 用国内镜像（清华 TUNA + npmmirror）；Playwright
#       浏览器走官方 CDN（实测 npmmirror 同步不全），见下方 ARG。
# 可选加固：用摘要锁定基础镜像，防上游被篡改
#   FROM nikolaik/python-nodejs:python3.13-nodejs26@sha256:<digest>
#   （docker manifest inspect nikolaik/python-nodejs:python3.13-nodejs26 获取）
# ============================================================================
FROM nikolaik/python-nodejs:python3.13-nodejs26

# 注意：RUN 指令的反斜杠续行中，注释必须独占一行（Dockerfile 解析器会
# 先剔除注释行再拼接命令）。行尾注释会进入 shell 导致构建失败。
ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

# ============================================================================
# 国内镜像源配置（构建时可用 --build-arg 覆盖；换回官方源见文末说明）
# ============================================================================
ARG APT_MIRROR="https://mirrors.tuna.tsinghua.edu.cn"
ARG PYPI_MIRROR="https://pypi.tuna.tsinghua.edu.cn/simple"
ARG NPM_MIRROR="https://registry.npmmirror.com"
# Playwright 浏览器特意用官方 CDN，不用 npmmirror：实测 npmmirror 对该
# 二进制同步不完整（firefox 的 debian-13 构建缺失、chromium 多个 revision
# 404），会导致构建失败；而 cdn.playwright.dev 国内可直连且文件齐全。
ARG PLAYWRIGHT_MIRROR="https://cdn.playwright.dev"
# 时区：构建时默认 UTC（适配全球用户），运行时通过 OpenClaw 配置 docker.env.TZ 注入
ARG TZ=UTC

# ENV 形式写入，构建期和运行期的 pip / uv / npm 都走镜像源；Playwright
# 浏览器走官方 CDN（原因见上方 ARG 注释）。
# 浏览器装到 /opt/ms-playwright 而不是默认的 /root/.cache：后者只有 root
# 可读，沙箱以非 root 的 sandbox 用户运行时会访问不到。
ENV PIP_INDEX_URL=${PYPI_MIRROR} \
    UV_DEFAULT_INDEX=${PYPI_MIRROR} \
    NPM_CONFIG_REGISTRY=${NPM_MIRROR} \
    PLAYWRIGHT_DOWNLOAD_HOST=${PLAYWRIGHT_MIRROR} \
    PLAYWRIGHT_BROWSERS_PATH=/opt/ms-playwright

# Debian trixie 使用 deb822 新格式（/etc/apt/sources.list.d/debian.sources）。
# Node 是官方 tarball 解压到 /usr/local 的（已核实构建模板），不走 apt，
# 换源不影响 Node 环境。
RUN set -eux; \
    # 归档 debian 以外的源配置，防止 apt update 仍去访问海外源
    mkdir -p /etc/apt/sources.list.d/disabled; \
    find /etc/apt/sources.list.d -maxdepth 1 -type f ! -name 'debian.sources' \
        -exec mv -t /etc/apt/sources.list.d/disabled {} +; \
    rm -f /etc/apt/sources.list; \
    { \
      echo "Types: deb"; \
      echo "URIs: ${APT_MIRROR}/debian"; \
      echo "Suites: trixie trixie-updates"; \
      echo "Components: main contrib non-free non-free-firmware"; \
      echo ""; \
      echo "Types: deb"; \
      echo "URIs: ${APT_MIRROR}/debian-security"; \
      echo "Suites: trixie-security"; \
      echo "Components: main contrib non-free non-free-firmware"; \
    } > /etc/apt/sources.list.d/debian.sources

# ============================================================================
# 先升级系统已有软件包（基础镜像可能有安全补丁未跟进），再安装新包
# DEBIAN_FRONTEND=noninteractive 防止升级时弹配置交互
# ============================================================================
RUN apt-get update \
    && apt-get upgrade -y --no-install-recommends \
    && apt-get install -y --no-install-recommends \
    # --- 基础 / agent 常用 ---
    git openssh-client ca-certificates \
    jq yq tree less file nano rsync \
    zip unzip bzip2 xz-utils 7zip \
    procps iputils-ping htop tmux fzf \
    # --- 编译依赖（python3-dev/build-essential 不需要：基础镜像的
    #     /usr/local Python 自带头文件与 gcc/g++/make）---
    libffi-dev libssl-dev pkg-config \
    # --- 多媒体 ---
    ffmpeg imagemagick \
    # --- 数据库客户端（不用可删，省约 100MB）---
    sqlite3 postgresql-client default-mysql-client redis-tools \
    # --- 网络诊断 ---
    openssl net-tools dnsutils curl wget \
    # --- Java 运行时（Jenkins CLI 需要）---
    default-jre-headless \
    # --- 现代 CLI（ripgrep 是 OpenClaw 生态标配）---
    ripgrep fd-find bat \
    # --- 文档处理：pandoc 格式转换，poppler-utils 提供 pdftotext ---
    pandoc poppler-utils \
    # --- 时区数据（显式配置，不依赖容器运行时的宿主机挂载）---
    tzdata \
    # --- GitHub CLI（trixie 官方源自带）---
    gh \
    # --- 浏览器反检测：Xvfb 虚拟显示器 + 中文字体 + 标准字体 ---
    # Xvfb 让 Chromium 以 headful 模式渲染到内存帧缓冲，消除 headless 指纹
    xvfb xauth dbus-x11 \
    fonts-noto-cjk fonts-noto-color-emoji fonts-liberation \
    fontconfig libfontconfig1 libfreetype6 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* \
    # Debian 把 fd/bat 重命名为 fdfind/batcat，补回通用名，
    # 否则 agent 习惯性调用 fd / bat 会 command not found
    && ln -sf /usr/bin/fdfind /usr/local/bin/fd \
    && ln -sf /usr/bin/batcat /usr/local/bin/bat

# ============================================================================
# GitLab CLI (glab)：Debian trixie 官方源暂无，从 GitLab 官方下载 deb 安装
# 用 ARG GLAB_VERSION 控制版本，升级时只改这一层不影响 apt 缓存
# ============================================================================
ARG GLAB_VERSION=1.113.0
RUN arch="$(dpkg --print-architecture)" \
    && curl -fsSL --proto '=https' \
       "https://gitlab.com/gitlab-org/cli/-/releases/v${GLAB_VERSION}/downloads/glab_${GLAB_VERSION}_linux_${arch}.deb" \
       -o /tmp/glab.deb \
    && dpkg -i /tmp/glab.deb \
    && rm -f /tmp/glab.deb \
    && glab --version

# ============================================================================
# kubectl：Kubernetes 命令行工具
# ============================================================================
ARG KUBECTL_VERSION=v1.36.3
RUN curl -fsSL --proto '=https' \
      "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/$(dpkg --print-architecture)/kubectl" \
      -o /usr/local/bin/kubectl \
    && chmod +x /usr/local/bin/kubectl \
    && kubectl version --client

# ============================================================================
# Grafana CLI：Grafana 服务端二进制自带 cli 子命令，下载 OSS tarball 后只取二进制
# ============================================================================
ARG GRAFANA_VERSION=13.1.3
RUN mkdir -p /tmp/grafana \
    && curl -fsSL --proto '=https' \
      "https://dl.grafana.com/oss/release/grafana-${GRAFANA_VERSION}.linux-$(dpkg --print-architecture).tar.gz" \
      -o /tmp/grafana.tar.gz \
    && tar -xzf /tmp/grafana.tar.gz -C /tmp/grafana --strip-components=1 \
    && cp /tmp/grafana/bin/grafana /usr/local/bin/grafana \
    && chmod +x /usr/local/bin/grafana \
    && rm -rf /tmp/grafana* \
    && grafana --version

# ============================================================================
# Jenkins CLI 包装脚本：jenkins-cli.jar 需从具体 Jenkins 实例下载，镜像构建时
# 无法固定版本。首次使用时从 $JENKINS_URL 自动拉取，缓存在 /tmp。
# 用法：jenkins-cli -s http://jenkins:8080 help
# ============================================================================
RUN printf '%s\n' \
    '#!/bin/bash' \
    'JAR="${JENKINS_CLI_JAR:-/tmp/jenkins-cli.jar}"' \
    'if [ ! -f "$JAR" ]; then' \
    '  : "${JENKINS_URL:?请设置 JENKINS_URL 环境变量，例如 http://jenkins:8080}"' \
    '  echo "Downloading jenkins-cli.jar from $JENKINS_URL..." >&2' \
    '  curl -fsSL "$JENKINS_URL/jnlpJars/jenkins-cli.jar" -o "$JAR"' \
    'fi' \
    'exec java -jar "$JAR" "$@"' \
    > /usr/local/bin/jenkins-cli && chmod +x /usr/local/bin/jenkins-cli

# ============================================================================
# 时区配置：构建时用 ARG TZ（默认 UTC）设置 /etc/localtime 作为兜底；
# 运行时 docker run -e TZ=Asia/Shanghai 可覆盖，无需重建镜像。
# DEBIAN_FRONTEND=noninteractive 防止 tzdata 安装时弹交互式选择。
# LANG/LC_ALL=C.UTF-8 保证中文文件名、输出和排序正确。
# ============================================================================
RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime \
    && echo $TZ > /etc/timezone

# ============================================================================
# OpenClaw 沙箱用户（与默认镜像 openclaw-sandbox:bookworm-slim 一致）
# 非 root 运行，缩小容器内爆炸半径
# ============================================================================
RUN useradd --uid 501 --create-home --shell /bin/bash sandbox \
    && mkdir -p /opt/ms-playwright \
    && chown -R sandbox:sandbox /opt/ms-playwright

# ============================================================================
# Python 包：分层利用 Docker 缓存
#   层 1：单独升级 uv（几乎不变，缓存命中率最高）
#   层 2：COPY requirements.txt 后用 uv 安装（uv 解析+并发下载比 pip 快；
#          requirements.txt 不变时整层命中缓存）。使用 BuildKit cache mount
#          保留 uv 下载缓存，改少量包时不必重新下载全部。
#   层 3：Playwright 浏览器（体积大、变动少，独立一层，不被 Python 包
#          改动牵连）
# 未钉版本 = 每次重建拿最新，可复现性差；如需稳定，在 requirements.txt
# 给核心包加约束，如 "pandas>=2.2,<3"。
# ============================================================================
RUN --mount=type=cache,target=/root/.cache/uv \
    python -m pip install --no-cache-dir uv

COPY requirements.txt /tmp/requirements.txt
RUN --mount=type=cache,target=/root/.cache/uv \
    uv pip install --system -r /tmp/requirements.txt

# Playwright 浏览器：必须用 python -m 调用（与 pip 安装的版本匹配；
# npx 走 npm 版本，两者不一致会导致启动报错）。
# --with-deps 自动安装系统依赖，能正确处理 trixie 的包名变更
# （如 libasound2 已改名 libasound2t64，手写包列表会构建失败）。
# firefox 约多占 350MB，不需要可去掉。
RUN python -m playwright install --with-deps chromium firefox \
    && chown -R sandbox:sandbox /opt/ms-playwright \
    # --with-deps 内部又跑了一次 apt-get update/install，会在本层重新
    # 生成 apt 缓存，必须在同一层清掉，否则白占几十 MB
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# ============================================================================
# Xvfb 虚拟显示器：启动时后台跑 Xvfb，让 Chromium/Firefox 能以 headful 模式
# 运行（渲染到内存帧缓冲，不需要真实显示器）。
#   DISPLAY=:99 由 /etc/profile.d/ 注入，所有 shell 自动继承。
#   分辨率默认 1920x1080x24，可用 XVFB_SCREEN 环境变量覆盖。
# ============================================================================
RUN printf '%s\n' \
    '#!/bin/bash' \
    'SCREEN="${XVFB_SCREEN:-1920x1080x24}"' \
    '# 确保 X11 临时目录存在（容器首次启动时可能缺失）' \
    'mkdir -p /tmp/.X11-unix && chmod 1777 /tmp/.X11-unix' \
    '# 清理可能残留的锁文件' \
    'rm -f /tmp/.X99-lock /tmp/.X11-unix/X99 2>/dev/null || true' \
    'exec Xvfb :99 -screen 0 "$SCREEN" -ac -nolisten tcp -noreset +extension RANDR' \
    > /usr/local/bin/xvfb-start && chmod +x /usr/local/bin/xvfb-start

# 登录 shell 自动启动 Xvfb 并设置 DISPLAY
RUN printf '%s\n' \
    '# Auto-start Xvfb for headful browsers (OpenClaw sandbox)' \
    'if [ -z "$DISPLAY" ] && command -v Xvfb >/dev/null 2>&1; then' \
    '  if ! pgrep -f "Xvfb :99" >/dev/null 2>&1; then' \
    '    /usr/local/bin/xvfb-start >/tmp/xvfb.log 2>&1 &' \
    '    # 等待 Xvfb 就绪（最多约 2 秒）' \
    '    for i in $(seq 1 20); do' \
    '      [ -e /tmp/.X11-unix/X99 ] && break' \
    '      sleep 0.1' \
    '    done' \
    '  fi' \
    '  export DISPLAY=:99' \
    'fi' \
    > /etc/profile.d/xvfb.sh && chmod +x /etc/profile.d/xvfb.sh

# Playwright 浏览器路径写进 profile，防止 su - / 登录 shell 重置环境后找不到
RUN echo 'export PLAYWRIGHT_BROWSERS_PATH=/opt/ms-playwright' \
    > /etc/profile.d/playwright.sh && chmod +x /etc/profile.d/playwright.sh

# ============================================================================
# 构建期自检：问题在构建时暴露，不留到运行时
# ============================================================================
RUN id sandbox \
    && python -c "import pandas, numpy, polars, playwright; print('python deps OK')" \
    && python -c "import playwright_stealth; print('playwright-stealth OK')" \
    && fd --version \
    && bat --version \
    && rg --version \
    && python -m playwright --version \
    && Xvfb -help >/dev/null 2>&1 && echo 'Xvfb OK' \
    && fc-list :lang=zh | head -1 | grep -q . && echo 'CJK fonts OK' \
    && test "$(cat /etc/timezone)" = "UTC" && echo 'tzdata OK' \
    && locale charmap | grep -qi UTF-8 && echo 'UTF-8 locale OK' \
    # su - 启动登录 shell，会自动 source /etc/profile.d/ 下的 xvfb.sh 和
    # playwright.sh，无需手动 export DISPLAY 或 PLAYWRIGHT_BROWSERS_PATH
    && su - sandbox -s /bin/bash -c \
       'python -c "from playwright.sync_api import sync_playwright; p=sync_playwright().start(); b=p.chromium.launch(headless=False); pg=b.new_page(); pg.set_content(\"<h1>test</h1>\"); print(\"headful chromium OK, title:\", repr(pg.title())); b.close(); p.stop()"'

# ============================================================================
# 运行时身份与启动命令
# OpenClaw 沙箱容器需要常驻（sleep infinity），由 OpenClaw 通过 docker exec
# 注入命令；bash 无 TTY 会立即退出导致容器停止
# ============================================================================
USER sandbox
WORKDIR /home/sandbox

CMD ["sleep", "infinity"]

# ============================================================================
# 换回官方源（在海外网络环境构建时）：
#   docker build \
#     --build-arg APT_MIRROR="http://deb.debian.org" \
#     --build-arg PYPI_MIRROR="https://pypi.org/simple" \
#     --build-arg NPM_MIRROR="https://registry.npmjs.org" \
#     -t aosolao/openclaw-sandbox:py313-node26-trixie .
# 说明：RUN 内会自动拼接 ${APT_MIRROR}/debian 与 ${APT_MIRROR}/debian-security，
# 与官方 deb.debian.org 的路径结构一致。Playwright 浏览器源默认已是官方
# CDN，无需处理。
# ============================================================================
