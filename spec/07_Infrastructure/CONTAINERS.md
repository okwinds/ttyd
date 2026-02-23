# 容器规格（Docker）

> 本文定义 `Dockerfile` 与 `Dockerfile.alpine` 的镜像结构与运行行为。  
> 事实源：`Dockerfile`、`Dockerfile.alpine`、`.github/workflows/docker.yml`。

---

## 1. 镜像 A：Ubuntu 20.04（`Dockerfile`）

事实源：`Dockerfile`。

- base：`ubuntu:20.04`
- build arg：`TARGETARCH`（由 buildx 注入）
- 依赖：
  - `apt-get install -y --no-install-recommends tini`
- 二进制复制：
  - `COPY ./dist/${TARGETARCH}/ttyd /usr/bin/ttyd`
  - 要求：构建上下文内存在对应目录（由 CI 先 cross-build 生成 `dist/<arch>/ttyd`）
- 端口：`EXPOSE 7681`
- 工作目录：`/root`
- 入口：
  - `ENTRYPOINT ["/usr/bin/tini", "--"]`
  - `CMD ["ttyd", "-W", "bash"]`

> 复刻一致性要点：默认 CMD 必须为 `ttyd -W bash`，这决定镜像启动后提供一个可写的 bash 终端。

---

## 2. 镜像 B：Alpine（`Dockerfile.alpine`）

事实源：`Dockerfile.alpine`。

- base：`alpine`
- build arg：`TARGETARCH`
- 依赖：
  - `apk add --no-cache bash tini`
- 二进制复制：同 Ubuntu 镜像
- 端口：`7681`
- 工作目录：`/root`
- 入口：
  - `ENTRYPOINT ["/sbin/tini", "--"]`
  - `CMD ["ttyd", "-W", "bash"]`

---

## 3. CI 中的多架构构建约定

事实源：`.github/workflows/docker.yml:20-27`。

- CI 会对 `amd64 armv7 arm64 s390x` 进行 cross-build；
- 目录名映射：
  - `armv7` 的 `TARGETARCH` 目录名会被改为 `arm`
- 最终用于镜像 build 的目录结构为：
  - `dist/amd64/ttyd`
  - `dist/arm/ttyd`
  - `dist/arm64/ttyd`
  - `dist/s390x/ttyd`

---

## 4. 运行时验收（容器）

复刻实现应通过以下验收：

1) `docker run -p 7681:7681 <image>` 启动后：
   - 浏览器访问 `http://localhost:7681/` 可看到终端页面；
2) WS 能连通，输入可写（默认 `-W`）；
3) `tini` 作为 PID 1，能正确转发信号并回收子进程。

