假如你修改了代码,比如阶段一,那我们开始学会怎么本地构建打包上传部署
阶段一：本地修改与构建（在项目根目录进行）
1. 修改前端代码（换皮）
直接修改 /web 目录下的 React 代码、Logo 图片等。改完保存，不需要手动编译。

2. 核心修改：使用国内镜像加速构建您的专属镜像
打开终端，在 ragflow 根目录（有 Dockerfile 的地方）执行以下命令：

Bash
docker build --build-arg NEED_MIRROR=1 -t lz-ragflow:v1.0 .
(注意：加上了 --build-arg NEED_MIRROR=1 后，它会自动使用阿里云等国内源，能帮你把打包时间从几小时缩短到十几分钟，并大幅减少报错率。)

阶段二：本地配置与全面测试（在 /docker 目录进行）
3. 修改环境配置 (.env)
进入 docker 目录，修改 .env 文件：

RAGFLOW_IMAGE=lz-ragflow:v1.0 (使用你刚打好的镜像)

DEVICE=gpu (开启你的 RTX 显卡加速，触发底层运行库下载的关键)

4. 拉取其他基础依赖

Bash
docker compose pull mysql minio redis es01
5. 启动并在本地跑通（⚠️ 核心步骤）

Bash
docker compose up -d
注意： 此时如果是首次启动，系统会在后台下载大量 GPU 运行库（PyTorch、CUDA等，约 2~3GB）。
请通过 docker logs -f docker-ragflow-gpu-1 观察日志，直到完全启动并提示就绪。然后访问 http://127.0.0.1 彻底测试。

🛑 【危险警告】 测试没问题后，绝对不要执行 docker compose down，否则刚才下载的几个 G 的依赖包会被清空！请直接进入阶段三！

阶段三：环境固化与离线导出（在终端执行）
6. 固化运行时容器（打快照）
为了让这 3GB 的运行库也能一起带到断网的 Linux 服务器上，我们需要把当前“吃饱了”的容器冻结成一个新镜像：

Bash
docker commit docker-ragflow-gpu-1 lz-ragflow:v1.1-full
(提示：打快照可能需要几十秒钟。完成后，你可以放心执行 docker compose down 停止本地服务了。)

7. 导出所有的 Docker 镜像
除了基础的 MySQL、Redis 等，由于你开启了 GPU 和本地测试，你需要注意打包 TEI (文本嵌入模型) 镜像。
执行以下命令（请根据你 docker images 查看到的实际 TAG 版本号为准）：

Bash
# 1. 导出你包含了所有 GPU 依赖的“终极离线镜像”
docker save -o lz-ragflow-full.tar lz-ragflow:v1.1-full

# 2. 导出周边所有依赖镜像
docker save -o dependencies.tar \
  mysql:8.0.39 \
  pgsty/minio:RELEASE.2026-03-25T00-00-00Z \
  valkey/valkey:8 \
  elasticsearch:8.11.3 \
  infiniflow/text-embeddings-inference:1.8
阶段四：服务器上传与最终上线（在 Linux 服务器进行）
8. 上传与导入
将 2 个 .tar 离线包和修改后的整个 docker 文件夹上传到服务器。然后在服务器执行：

Bash
docker load -i dependencies.tar
docker load -i lz-ragflow-full.tar
9. 修改线上配置与一键启动
在服务器上进入上传的 docker 文件夹，
在服务器上进入上传的 docker 文件夹
注意:docker下面有两个.sh文件.要使用命令
 chmod +x migration.sh 
 chmod +x entrypoint.sh
 添加权限后
打开 .env 文件，做最后一次确认：

必须修改： RAGFLOW_IMAGE=lz-ragflow:v1.1-full (确保使用的是快照版)

确认 DEVICE 适配了服务器硬件（有显卡就 gpu，没显卡改回 cpu）

确认 USE_DOCLING=false（如果不依赖它的话，建议写死关闭）

保存后执行：

Bash
docker compose up -d
恭喜！真正的 0 下载、纯净离线版专属知识库正式上线！