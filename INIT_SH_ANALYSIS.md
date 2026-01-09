# init.sh 修改分析

## 问题分析

init.sh 是Docker启动脚本，包含约12处硬编码路径，**必须修改**才能在AlwaysData上正常运行。

### hardcode 路径列表

| 行号 | 路径 | 用途 |
|------|------|------|
| 3 | `/opt/alist/data/.init` | 初始化标记文件 |
| 9 | `/opt/atv/BOOT-INF/lib/h2-*.jar` | H2数据库驱动 |
| 15 | `/data/atv` 等 | Docker内数据目录 |
| 29 | `/opt/alist/data/config.json` | AList配置文件 |
| 30 | `/opt/alist/data/config.json` | AList配置文件 |
| 33 | `/opt/alist/alist admin` | AList管理命令 |
| 46 | `/opt/atv/data/data` | 数据库文件路径 |
| 50 | `/opt/atv/BOOT-INF/lib/h2-2.3.232.jar` | H2新版驱动 |
| 56 | `/data/h2.version.txt` | H2版本标记 |

---

## 修改方案

### 方案选择

init.sh 是Docker专用脚本，**不应修改为支持ALIST_HOME**，原因：

1. **Docker环境特殊性**
   - Docker内的路径是固定的（/data, /www, /opt/alist等）
   - 不涉及AlwaysData部署（AlwaysData直接运行Java，不用init.sh）
   - Docker内的/opt/atv只是镜像内部的临时路径

2. **使用场景不同**
   - init.sh 只在Docker容器启动时执行一次
   - AlwaysData 是独立部署，直接运行Java应用
   - 两者的初始化流程完全不同

3. **修改风险**
   - 改错了会破坏现有Docker部署
   - init.sh涉及复杂的初始化逻辑（数据库升级、资源解压等）

---

## 正确的实施方案

### AlwaysData 初始化流程

```
AlwaysData 独立部署（不使用init.sh）
         ↓
1. 用户设置 export ALIST_HOME=/home/v587xxoo/alist
         ↓
2. 运行 alwaysdata-deploy.sh 创建目录结构
         ↓
3. 配置 application.yaml
         ↓
4. 直接 java -jar target/alist-tvbox-1.0.jar
         ↓
应用自动使用 ALIST_HOME 作为基础路径
```

### Docker 初始化流程（保持不变）

```
Docker 构建和启动（保持原有逻辑）
         ↓
1. docker run ... /entrypoint.sh
         ↓
2. entrypoint.sh 执行 init.sh
         ↓
3. init.sh 创建Docker内的初始化结构（/data, /www等）
         ↓
4. 启动Java应用（检测到inDocker=true，使用/data等）
         ↓
应用正常运行
```

---

## 总结

### 修改清单（更新）

| 文件 | 是否修改 | 原因 |
|------|--------|------|
| Utils.java | ✅ 必须 | 核心路径处理 |
| AListLocalService.java | ✅ 必须 | 日志路径 |
| IndexService.java | ✅ 必须 | 版本路径 |
| LogsService.java | ✅ 必须 | 日志目录 |
| application.yaml | ✅ 建议 | 配置支持 |
| install-service.sh | ✅ 可选 | 本地部署脚本 |
| **init.sh** | ❌ **不改** | **Docker专用，不影响AlwaysData** |

### 关键点

```
✅ AlwaysData 不使用 init.sh
✅ init.sh 只在Docker内执行
✅ Docker 内的路径不受 ALIST_HOME 影响
✅ 修改 Utils.java 自动覆盖所有非Docker场景
```

所以修改计划 **不需要** 包括 init.sh。
