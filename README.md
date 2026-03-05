# 🛠️ 宝塔 + Docker 部署 Dujiao-Next (新版独角) 教程

感谢独角（Dujiao-Next）作者 **assimon** 的开源项目！本篇教程将指导你使用宝塔国际版（aaPanel）结合 Docker 环境，优雅地部署新版独角。

相比于社区文档中使用 aaPanel 手动下载编译好的二进制文件部署，本篇采用的 **宝塔+Docker部署** 方案，在后续的版本更新维护上会更加方便快捷。

#### 注意⚠️
**本篇是宝塔Docker部署教程，和1Panel有所不同！请不要弄混了docker-compose.postgres.yml和config.yml文件！**

**1Panel面板部署教程仓库在：[![GitHub Repo](https://img.shields.io/github/repo-size/qukuan/1panel-dujiao-next?logo=github&label=Repo)](https://github.com/qukuan/1panel-dujiao-next)

---

#### 📷 视频教程（保姆级）一看就会

[![Bilibili](https://img.shields.io/badge/Bilibili-国内网点击观看blibli视频教程-blue?style=for-the-badge&logo=bilibili)](https://b23.tv/gcUkVec)

[![YouTube Video](https://img.shields.io/badge/Watch%20Video-YouTube-brightgreen)](https://youtu.be/741xESqsA7s?si=Td9cNQZoBZs1QlE9)

---

## 📌 准备工作与参考链接

*   **服务器 (VPS)**：推荐使用 Ubuntu 24.04 系统。
*   **面板环境**：安装宝塔国际版 [aaPanel](https://www.aapanel.com/)（由官方更新维护，可选中文，无需绑定邮箱或手机号）。
*   **相关仓库与文档**：
    *   新独角 (Dujiao-Next) 官方 Github：[dujiao-next/dujiao-next](https://github.com/dujiao-next)
    *   社区搭建文档：[dujiao-next.com](https://dujiao-next.com/)
    *   本教程及 1Panel 版本参考仓库：[qukuan/1panel-dujiao-next](https://github.com/qukuan/1panel-dujiao-next) (1panel 的教程已有视频版本)。

> ⚠️ **血泪警告**：
> 千万千万不要去安装网上所谓的“开心版”、“破解版”面板！！！到时候真被爆库或挂马，那就真的是“开心”了！！！

---

## 🚀 开始部署

### 1. 软件安装与准备

前往宝塔（aaPanel）的应用商店，下载以下环境：
*   **Nginx**：版本无特殊要求。
*   **Redis**：版本无特殊要求。
*   **PostgreSQL**：版本看具体情况。
    *   *注意：每台 VPS 安装过程可能不同。如果安装最新版 postgres Manager 2.8 失败，建议切换回 Manager 2.5 版本即可。*
*   **Docker**：宝塔不自带 Docker（1Panel是自带的），需要手动点击侧边栏的 **Docker** 菜单，第一次点击会提示你安装，点击一键安装即可。

### 2. 创建 PostgreSQL 数据库

等待前面的 postgres Manager 2.5 安装完成后：
1. 点击宝塔左侧菜单：**数据库**。
2. 切换为 **PgSQL** 选项卡。它会提示你选择版本并安装数据库，选择 **16 版本** 即可（18 最新版本可能会报错，视情况而定）。
3. 安装完毕后，创建一个新数据库。
4. **务必牢记**：你自己创建的 `用户名`、`数据库名称` 和 `密码`。

### 3. 配置 PostgreSQL 允许容器连接

为了让 Docker 容器能连上宝塔安装的数据库，需要修改配置：
1. 打开 PgSQL 的 **Configuration file**，在最后一行添加：
   ```ini
   listen_addresses = '*'
   ```
   *添加后点击 Save 保存。*

2. 打开 **Client authentication**，在最后添加你的宝塔容器网络段（允许容器项目连接到宝塔应用安装的 PostgreSQL）：
   ```text
   host    all             all             172.17.0.0/16           md5
   host    all             all             172.18.0.0/16           md5
   host    all             all             172.19.0.0/16           md5
   ```
   *添加后点击 Save 保存。*

3. 最后来到 **Service Status**：
   * 先点击重新加载：**Reload**
   * 再点击重启：**Restart**

> 💡 **如何查看这三个容器网络 IP？**
> 安装好 Docker 后，点击宝塔菜单 **容器** -> **总览** -> **网络**。即可看到自己的容器网络网段。如果和上面的 IP 不一样，请自行替换。
> 🚨 **安全建议**：上线生产环境不建议全部放行（不建议开启允许外部访问，也就是设置成 `0.0.0.0`）。

### 4. 补充 Redis 配置

安装好 Redis 后，再去宝塔的应用商店，点击已安装的 Redis，继续点击 **优化**：

1. **`bind` 配置项**：补充上我们的容器网络 IP。
   ```text
   # 例如：
   bind 127.0.0.1 172.17.0.1 172.18.0.1 172.19.0.1
   ```
   *(注意 IP 之间的空格！意思就是允许本地服务器连接、允许自己的容器项目连接。一样不建议生产环境设置为 0.0.0.0！)*

2. **`requirepass` 配置项**：设置一个 Redis 密码。
   * 自定义填写一个密码。生产环境 **十分建议设置**！多一层保护。

完成这两项设置，点击 **Save** 保存。
**注意**：保存成功后，再点击一下 **重启 Redis** 立即生效。

### 5. 添加站点

分别为 `user` 用户前端、`admin` 管理前端、`api` 后端，**添加三个站点**。

配置步骤：
* 配置 SSL 证书并开启强制 HTTPS。
* 设置反向代理监听端口。
* 补充反向代理配置。
*(注：1panel 和宝塔的反向代理配置信息可能不同，请根据实际情况灵活配置)*

[补充反向代理配置信息](https://github.com/qukuan/1panel-dujiao-next/tree/main/nginx)

### 6. 端口放行

点击宝塔菜单：**安全**，添加放行以下端口：
* 你为 `user前端`、`admin前端`、`api后端` 设置的 **三个反代监听端口**。
* **Redis 端口**：`6379` 
* **PostgreSQL 端口**：`5432` 
*(如果还没放行的话请放行，视内网互通需求而定)*

### 7. 准备部署目录

您可以选择使用 SSH 连接终端执行命令，或者直接在 1Panel/宝塔 面板中可视化创建。

#### 方式一：终端命令创建 (推荐)
依次复制以下命令并回车执行：

```bash
mkdir -p /www/dujiao-next/{config,data/db,data/uploads,data/logs,data/redis,data/postgres}
```
```bash
cd /www/dujiao-next
```
```bash
chmod -R 0777 ./data/logs ./data/db ./data/uploads ./data/redis ./data/postgres 
```

#### 方式二：在宝塔面板手动创建相关文件夹
1. 进入 `/www` 文件夹 → 创建文件夹 `dujiao-next`
2. 在 `dujiao-next` 文件夹内继续创建以下文件夹：`config` 和 `data`
3. 进入 `data` 文件夹内，创建以下子目录：`logs`、`postgres`、`redis`、`uploads`
4. 返回根目录（即 `dujiao-next`），全选 `config` 和 `data` 文件夹 → 点击 **权限** → 设置为 **777**。

> 📌 **提示**：文件路径仅作为参考，步骤和路径是死的，人是活的，都可以自定义监听端口号、项目路径等。

### 8. 上传并配置核心文件

将以下主要三个文件上传到刚才创建的目录中：

*   `.env` (放在项目根目录下)
*   `docker-compose.postgres.yml` (放在项目根目录下)
*   `config.yml` (放在 `config` 文件夹内)

**修改配置信息**：
双击打开 `.env` 和 `config.yml` 文件填写自己的配置信息。
* **数据库配置**：数据库用户、数据库名称、数据库密码，这三项填自己的，其他默认即可。
* **注意**：填写 `config.yml` 配置的时候，严格注意 YAML 语法的 **空格**！

### 9. 启动与运行

确保当前在 `/www/dujiao-next` 目录下，打开终端，执行启动命令：

```bash
docker compose --env-file .env -f docker-compose.postgres.yml up -d
```

---

## ❓ 遇到的问题排查 (FAQ)

我想很多人都可能会遇到 API 容器启动失败、报错、或者始终卡在 API 启动上。

**💡 终极排错法宝**：
查看具体的报错日志！去 `data/logs/` 文件夹内查看 `app.log` 文件，或者点击宝塔菜单：**容器 -> API 容器日志**。

---

### Q1：后台账号密码正确，登录提示：“限流服务不可用”？
**A**：基本是 Redis 没有连接成功，导致 API 没有跑起来。重点检查 Redis 相关配置（如 bind 和 密码）。

### Q2：容器全部启动成功，启动时也没有报错，账号密码正确，登录时提示：“405 错误”？
**A**：
1. **跨域配置没填对**：也就是 `config.yml` 文件当中的 `CORS` 跨域配置。一个填的是你的 `user` 用户前端域名，另一个应该是 `admin` 后台前端域名。
2. **反向代理没设置好**：`admin` 后台前端的反向代理配置可能没设置正确。

> 🤬 **千万记住**：
> 不要总开口就说：“都对啊，设置都对啊”——这是自信产生的错觉！
> **对几把毛，对就不会报错了！！！**

---

## 🎉 登录后台

我刚刚部署成功并登录过后台，由于 Redis 缓存还没失效，所以再次打开会自动登录。

🚨 **第一件要做的事**：
登录后台后，**立刻更改弱密码（默认密码）**！

本篇教程是宝塔+Docker部署，后续更新版本也会非常方便。快去享受你的新版独角吧！





