# AlwaysData 部署修改计划

## 一、现状分析

### 1.1 现有硬编码路径统计

| 位置 | 文件 | 行数 | 现状 |
|------|------|------|------|
| Java代码 | Utils.java | 279,284,289,294,302 | 5处hardcode路径 |
| Java代码 | AListLocalService.java | 65 | 1处hardcode AList日志路径 |
| Java代码 | IndexService.java | 162,178 | 2处hardcode版本文件路径 |
| Java代码 | LogsService.java | 126,133 | 2处hardcode日志目录 |
| 配置文件 | application.yaml | 9,37 | 2处hardcode数据库和日志路径 |
| 配置文件 | application-docker.yaml | 3 | 1处hardcode数据库路径 |
| 配置文件 | application-xiaoya.yaml | 3,9 | 2处hardcode数据库路径 |
| Shell脚本 | install-service.sh | 多处 | 约15处hardcode |
| Shell脚本 | init.sh | 多处 | 约10处hardcode |

**总计**: 约40+处hardcode路径

---

## 二、修改策略

### 2.1 核心理念

使用 **`ALIST_HOME` 环境变量** 替代硬编码的 `/opt/atv/` 前缀：

```bash
# 用户设置环境变量
export ALIST_HOME=/home/v587xxoo/alist

# 系统自动转换
/opt/atv/data     →  $ALIST_HOME/data
/opt/atv/log      →  $ALIST_HOME/log
/opt/atv/www      →  $ALIST_HOME/www
/opt/atv/alist    →  $ALIST_HOME/alist
/opt/atv/index    →  $ALIST_HOME/index
```

### 2.2 优势

| 优势 | 说明 |
|------|------|
| ✅ **无需修改源代码** | 仅修改配置和工具类 |
| ✅ **向后兼容** | 不设置变量时默认用 `/opt/atv/` |
| ✅ **灵活部署** | 支持多个部署实例（不同HOME） |
| ✅ **环境隔离** | 每个用户独立HOME目录 |
| ✅ **易于维护** | 集中在Utils.java管理 |

---

## 三、详细修改计划

### 3.1 **阶段1: 修改 Utils.java（核心处理）**

**文件**: `src/main/java/cn/har01d/alist_tvbox/util/Utils.java`

**修改点**:

```java
// 第60-70行：添加ALIST_HOME处理
private static final String ALIST_HOME;

static {
    readUserAgents();
    inDocker = System.getenv("INSTALL") != null && Files.exists(Path.of("/entrypoint.sh"));
    
    // 新增：处理ALIST_HOME环境变量
    ALIST_HOME = System.getenv("ALIST_HOME") != null ? 
        System.getenv("ALIST_HOME") : 
        "/opt/atv";
    
    log.info("ALIST_HOME: {} (inDocker: {})", ALIST_HOME, inDocker);
}

// 修改所有路径获取方法（使用ALIST_HOME）
public static Path getDataPath(String... path) {
    String base = inDocker ? "/data" : ALIST_HOME + "/data";
    return Path.of(base, path);
}

public static Path getWebPath(String... path) {
    String base = inDocker ? "/www" : ALIST_HOME + "/www";
    return Path.of(base, path);
}

public static Path getIndexPath(String... path) {
    String base = inDocker ? "/data/index" : ALIST_HOME + "/index";
    return Path.of(base, path);
}

public static Path getLogPath(String name) {
    String base = inDocker ? "/data/log" : ALIST_HOME + "/log";
    return Path.of(base, name);
}

public static String getAListPath(String name) {
    if (name.startsWith("/")) {
        return name;
    }
    String base = inDocker ? "/opt/alist/" : ALIST_HOME + "/alist/";
    return base + name;
}
```

**影响范围**: 自动覆盖所有调用 getDataPath/getWebPath/等方法的地方

---

### 3.2 **阶段2: 修改 AListLocalService.java**

**文件**: `src/main/java/cn/har01d/alist_tvbox/service/AListLocalService.java`

**修改点**:

```java
// 第65行：修改硬编码的AList日志路径
// 原来：
private String aListLogPath = "/opt/alist/log/alist.log";

// 修改为：
private String aListLogPath = Utils.getAListPath("log/alist.log");
```

---

### 3.3 **阶段3: 修改 IndexService.java**

**文件**: `src/main/java/cn/har01d/alist_tvbox/service/IndexService.java`

**修改点**:

```java
// 第162行：修改app_version路径
// 原来：
Path path = Path.of(Utils.inDocker ? "/app_version" : "/opt/atv/data/app_version");

// 修改为：
Path path = Utils.getDataPath("app_version");

// 第178行：修改AList版本路径
// 原来：
Path path = Path.of("/opt/atv/alist/data/version");

// 修改为：
Path path = Utils.getAListPath("data/version");
```

---

### 3.4 **阶段4: 修改 LogsService.java**

**文件**: `src/main/java/cn/har01d/alist_tvbox/service/LogsService.java`

**修改点**:

```java
// 第126行：修改日志导出路径
// 原来：
FileUtils.copyFileToDirectory(file, new File("/opt/atv/log/"));

// 修改为：
FileUtils.copyFileToDirectory(file, Utils.getLogPath("").toFile());

// 第133行：修改日志压缩路径
// 原来：
File fileToZip = new File("/opt/atv/log/");

// 修改为：
File fileToZip = Utils.getLogPath("").toFile();
```

---

### 3.5 **阶段5: 修改配置文件**

#### 5.1 `application.yaml` - 使用占位符

**修改方案**：使用Spring的属性占位符

```yaml
spring:
  datasource:
    jdbc-url: jdbc:h2:file:${ALIST_DATA_PATH:file:/opt/atv/data/data}
    # 或使用自定义属性
    
app:
  data-path: ${ALIST_HOME:/opt/atv}

logging:
  file:
    name: ${ALIST_LOG_PATH:${ALIST_HOME:/opt/atv}/log/app.log}
```

#### 5.2 `application-docker.yaml` - 保持Docker模式

```yaml
spring:
  datasource:
    jdbc-url: jdbc:h2:file:/data/atv  # Docker中保持不变
```

#### 5.3 `application-xiaoya.yaml` - 保持Xiaoya模式

```yaml
spring:
  datasource:
    jdbc-url: jdbc:h2:file:/data/atv  # Xiaoya中保持不变
```

---

### 3.6 **阶段6: 修改应用配置类**

**文件**: `src/main/java/cn/har01d/alist_tvbox/config/AppProperties.java`

**添加属性**:

```java
@Data
@ConfigurationProperties("app")
public class AppProperties {
    // 现有属性...
    
    // 新增属性
    private String dataPath = System.getenv("ALIST_HOME") != null ? 
        System.getenv("ALIST_HOME") + "/data" : 
        "/opt/atv/data";
    
    // Getter/Setter会自动生成
}
```

---

### 3.7 **阶段7: 修改启动脚本**

#### 7.1 创建新脚本 `scripts/alwaysdata-deploy.sh`

```bash
#!/bin/bash

# AlwaysData 部署脚本
ALIST_HOME=${ALIST_HOME:-/home/v587xxoo/alist}
ALIST_USER=${ALIST_USER:-v587xxoo}

# 创建目录结构
mkdir -p "$ALIST_HOME"/{config,scripts,index,log,data/{atv,backup},www/{cat,pg,zx,tvbox,files},alist/{data,log}}

# 设置权限
chmod -R 755 "$ALIST_HOME"

# 导出环境变量
export ALIST_HOME

echo "AlwaysData deployment initialized:"
echo "  ALIST_HOME: $ALIST_HOME"
echo "  Directories created:"
echo "    - Data: $ALIST_HOME/data"
echo "    - Logs: $ALIST_HOME/log"
echo "    - Web: $ALIST_HOME/www"
echo "    - AList: $ALIST_HOME/alist"
echo "    - Index: $ALIST_HOME/index"
```

#### 7.2 修改 `install-service.sh`

在脚本开头添加：

```bash
#!/bin/bash

# 获取ALIST_HOME（优先级：脚本参数 > 环境变量 > 默认值）
ALIST_HOME="${1:-${ALIST_HOME:-/opt/atv}}"

# 后续所有 /opt/atv 替换为 $ALIST_HOME
mkdir -p "$ALIST_HOME"/{config,scripts,index,log,data/atv,data/backup,www/{cat,pg,zx,tvbox,files},alist/{data,log}}
chown -R ${USER}:${GROUP} "$ALIST_HOME"

# 配置文件路径也使用变量
conf="$ALIST_HOME/config/application-production.yaml"

# systemd服务配置
WorkingDirectory=$ALIST_HOME
ExecStart=$ALIST_HOME/atv --spring.profiles.active=standalone,production
```

---

## 四、实施步骤（推荐顺序）

### 🔧 **第一批：核心代码修改（优先级：高）**

```
1. Utils.java
   - 添加ALIST_HOME静态变量
   - 修改所有getXxxPath()方法
   - 添加日志输出ALIST_HOME值

2. AListLocalService.java
   - 修改aListLogPath

3. IndexService.java  
   - 修改两处版本文件路径

4. LogsService.java
   - 修改两处日志目录
```

**预期改动**: 4个文件，约10处改动，无功能变化

---

### 📝 **第二批：配置文件修改（优先级：中）**

```
1. application.yaml
   - 添加属性占位符支持ALIST_HOME
   
2. AppProperties.java
   - 添加dataPath属性（可选，用于动态访问）
```

**预期改动**: 2个文件，完全向后兼容

---

### 🚀 **第三批：部署脚本优化（优先级：中）**

```
1. 创建 alwaysdata-deploy.sh
   - 新部署脚本，专用于AlwaysData

2. 更新 install-service.sh
   - 支持ALIST_HOME参数
   - 替换所有硬编码路径
```

**预期改动**: 新增脚本 + 修改现有脚本

---

## 五、使用示例

### 5.1 AlwaysData 环境部署

```bash
# 1. 下载项目
cd /home/v587xxoo
git clone https://github.com/power721/alist-tvbox.git
cd alist-tvbox

# 2. 设置环境变量（添加到 ~/.bashrc）
export ALIST_HOME=/home/v587xxoo/alist

# 3. 构建项目
mvn clean package -DskipTests

# 4. 初始化目录
bash scripts/alwaysdata-deploy.sh $ALIST_HOME

# 5. 配置数据库（application.yaml）
# 创建 $ALIST_HOME/config/application.yaml

# 6. 运行应用
java -jar target/alist-tvbox-1.0.jar \
  --spring.config.location=file:$ALIST_HOME/config/application.yaml \
  --server.port=8080
```

### 5.2 Docker模式（保持原有）

```bash
# 不设置ALIST_HOME，自动使用/opt/atv
docker run -d \
  -p 4567:4567 \
  -v /data:/data \
  haroldli/xiaoya-tvbox:latest
```

### 5.3 本地开发（保持原有）

```bash
# 不设置ALIST_HOME，自动使用/opt/atv
java -jar target/alist-tvbox-1.0.jar
```

---

## 六、兼容性保证

### 6.1 向后兼容性

| 场景 | ALIST_HOME | 实际路径 | 备注 |
|------|-----------|---------|------|
| **Docker** | 不设置 | `/data` | inDocker=true时优先 |
| **本地开发** | 不设置 | `/opt/atv` | 默认行为不变 |
| **AlwaysData** | `/home/v587xxoo/alist` | `/home/v587xxoo/alist` | 新增场景 |
| **其他自定义** | `/custom/path` | `/custom/path` | 灵活配置 |

### 6.2 优先级顺序

```
1. ALIST_HOME 环境变量（用户设置）
2. inDocker检测（Docker/本地自动）
3. 默认值 /opt/atv（降级）
```

---

## 七、测试计划

### 7.1 单元测试

```java
// 测试Utils.getDataPath() 在不同ALIST_HOME下的行为
@Test
void testGetDataPathWithAlistHome() {
    System.setProperty("ALIST_HOME", "/custom/path");
    Path result = Utils.getDataPath("config.json");
    assertEquals(Path.of("/custom/path/data/config.json"), result);
}
```

### 7.2 集成测试

```bash
# 测试1：Docker模式
docker run -e INSTALL=native ... # inDocker=true, 使用/data

# 测试2：本地模式（未设置ALIST_HOME）
java -jar alist-tvbox-1.0.jar # 使用/opt/atv

# 测试3：AlwaysData模式（设置ALIST_HOME）
export ALIST_HOME=/home/v587xxoo/alist
java -jar alist-tvbox-1.0.jar # 使用/home/v587xxoo/alist
```

---

## 八、风险评估

### 8.1 低风险项

| 项目 | 风险 | 缓解措施 |
|------|------|--------|
| Utils.java修改 | 影响所有路径 | 充分测试path类所有方法 |
| 环境变量添加 | 多环境冲突 | 优先级文档清晰 |
| 脚本修改 | 命令执行失败 | 保留原脚本备份 |

### 8.2 测试覆盖

- ✅ 各种路径组合测试
- ✅ 目录权限测试  
- ✅ 文件读写测试
- ✅ 日志输出验证

---

## 九、预期工作量

| 阶段 | 文件数 | 行数 | 工时 |
|------|--------|------|------|
| 阶段1-4（代码修改） | 4 | 50-100 | 1-2h |
| 阶段5-6（配置修改） | 4 | 20-40 | 1h |
| 阶段7（脚本优化） | 2 | 100+ | 1-2h |
| **总计** | **10** | **170-240** | **3-5h** |

---

## 十、实施建议

### ✅ 建议做法

1. **逐阶段实施** - 先修改代码，再修改脚本
2. **充分测试** - 三种模式都要测试
3. **版本备份** - 提交到分支，不直接改main
4. **文档更新** - 更新README和部署指南
5. **回归测试** - 确保Docker/本地模式仍正常工作

### ❌ 避免做法

1. 不要一次性修改所有地方
2. 不要删除原有的硬编码常量
3. 不要改变现有的Docker/本地行为
4. 不要添加过多的配置项

---

## 总结

**核心方案**：通过 `ALIST_HOME` 环境变量实现灵活部署路径，修改点集中在 `Utils.java`，自动覆盖整个应用的路径依赖。

**优势**：
- ✅ 改动最小（仅4个Java文件+配置）
- ✅ 风险最低（单点修改）
- ✅ 兼容性好（向后兼容）
- ✅ 灵活性强（支持多环境）

**立即可做**：阶段1（Utils.java）可独立完成，其他阶段可选
