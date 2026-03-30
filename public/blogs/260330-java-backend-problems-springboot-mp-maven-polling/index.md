## 一、SpringBoot 核心基础&生命周期
### 1. MultipartFile 的作用

MultipartFile 是 spring 框架提供的接口，主要用于接受和处理通过 HTTP `multipart/form-data` 协议上传的文件。

主要方法如下：

+   `getName()` : 获取表单中文件输入框的 `name` 属性
+   **`getOriginalFilename()`** ：获取文件在客户端上的原始文件名（带后缀）。
+   `getContentType()`：获取文件的 MIME 类型（如 `image/jpeg` 或 `application/pdf`）。MIME 类型是文件的真实身份标签（大类/小类），比文件后缀名更靠谱，以为 MIME 类型是文件自带的，无法更改。
+   `isEmpty()`：判断文件是否为空（没有选择文件或是文件大小为0）
+   `getSize()` : 获取文件大小
+   `getBytes()` ：一次性将文件内容读取到字节数组中（可能会导致 OOM）
+   `getInputStream()` ： 获取输入流，用于流式读取，适合处理大文件
+   **`transferTo(File dest)`** ： 直接将上传的文件保存到指定的服务器本地路径。

**注意事项**：

在上传文件时需要设置单个文件的最大限制和单次请求的最大限制，否则上传文件过大会导致 spring 报错。配置如下：

```yaml
spring:
	servlet:
		multiport:
			max-file-size: 10MB      # 单个文件的最大限制
			max-request-size: 50MB   # 单个请求的总大小限制（包含多个文件）
```



### 2. CommandLineRunner 的作用

CommandLineRunner 是 spring 提供的接口，作用是在 spring boot 项目启动之后自动执行一段自定义的代码。

代码示例：

```java
@Component
public class MyStartupRunner implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
        // 项目启动成功后，这里的代码会自动执行
        System.out.println("Spring Boot 已启动，执行初始化任务...");
    }
}
```

**关键点**:

1.   必须加上 `@Component` ，让 spring 管理
2.   多个 `CommandLineRunner` 可以通过 `@Order(1)` 指定执行顺序
3.   方法执行的时机：容器完全初始化完毕，Bean都创建好之后
4.   抛出异常会导致项目启动失败

---

## 二、Maven项目构建问题
### 1. POM父路径指向错误导致项目结构校验失败

**报错信息**：

>   'parent.relativePath' of POM org.example:SmzPersonUpload_JBH:1.0-SNAPSHOT (F:\javaworkspace\dis-offline-system\dis-offline-system\pom.xml) points at org.example:SmzPersonUpload_JBH instead of org.springframework.boot:spring-boot-starter-parent, please verify your project structure。

**报错原因**：

Maven 发现你在当前的项目中声明了一个父工程（parent），但是实际路径下的工程和你声明的父工程并不是同一个工程

**解决方法**：

**核心就是将子模块中的父引用设置正确**。

+   方法一：直接将父工程声明中的相对路径改为 `<relativePath\>`，这会强制 Maven 直接去你的**本地仓库**（Local Repository）或远程仓库查找 Spring Boot 官方父级 POM，而不是在你的文件目录里瞎晃悠。
+   方法二：确认当前项目的 `pom.xml` 中 `<parent>` 标签内的 `groupId` 和 `artifactId` 是否真的是你想要的。如果该项目本身就是一个子模块，且父模块应该是 `SmzPersonUpload_JBH`，那么你需要修改 `<parent>` 节点的内容来匹配它。

---

## 三、ORM 框架（MyBatis-Plus）实操问题

### 1. MyBatis-Plus 数据库新增字段后如何快速更新实体 / mapper

使用 MyBatisX-Generator 重新生成即可，路径相同的话会直接覆盖掉原文件。这里有个问题是 MyBatisX-Generator 是可以生成 service 的，所以如果重新生成的话，新的 service 可能会覆盖掉之前写好的 service，所以使用的时候需要慎重。



----

## 四、分布式基础 & 算法原理

### 1.使用Hutool提供的雪花算法能够保证不同机器上生成的id不重复吗？

默认情况下，Hutool 无法百分之百保证不同物理机器上的 ID 不重复。

雪花算法的核心结构是：`符号位(1 位) + 时间戳(41 位) + 数据中心ID(datacenterId 5位) + 工作机器ID(workerId 5位) + 序列号(12 位)`。如果不告诉 Hutool 每台机器的 ID 是多少，它会默认 `datacenterId` 和 `workerId` 都是 `1`。此时两台机器同时调用 `IdUtil.getSnowflakeNextId()` 就有可能生成相同的ID。

解决方法：在配置类中，根据当前机器的硬件信息初始化这个工具。workerId(5位) 和 datacenterId(5位) 共同组成了机器ID。

```java
import cn.hutool.core.lang.Snowflake;
import cn.hutool.core.net.NetUtil;
import cn.hutool.core.util.IdUtil;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SnowflakeConfig {

    @Bean
    public Snowflake snowflake() {
        // 1. 获取本机的 MAC 地址 (例如 "00-16-EA-AE-3C-40")
        String macAddress = NetUtil.getLocalMacAddress();

        // 2. 如果获取不到 MAC（比如某些特殊虚拟机环境），保底给个随机数
        long macHash = macAddress != null ? macAddress.hashCode() : (long) (Math.random() * 1024);

        // 3. 将负数转正，避免取余报错
        macHash = Math.abs(macHash);

        // 4. 将 MAC 的 Hash 值分配给 workerId 和 datacenterId
        // workerId 取余 32 (占用 5 bit)
        long workerId = macHash % 32;
        // datacenterId 先除以 32 再取余 32 (占用另外 5 bit)
        long datacenterId = (macHash / 32) % 32;

        // 5. 注入 Hutool 的 Snowflake 实例
        return IdUtil.getSnowflake(workerId, datacenterId);
    }
}
```

在业务代码中使用：

```java
@Autowired
private Snowflake snowflake;

public void uploadFile(...) {
    long fileId = snowflake.nextId(); // 这样生成的 ID 在不同机器间就是唯一的
}
```



---

## 五、接口测试 & 工具实操

### 1. controller 请求参数中包含文件流，我在postman该如何模拟

1.   设置 Body：点击 **Body** 标签页，勾选 **`form-data`** 单选框。
2.   添加请求参数:
     1.   在 `Key` 这一列，输入 `file`（必须和 DTO 里的属性名完全一致）。**关键操作：** 将鼠标悬停在刚才输入的 `file` 单元格右侧，你会看到一个隐藏的下拉菜单（默认显示为 `Text`）。点击它，将其改为 **`File`**。这时，`Value` 这一列会出现一个 **“Select Files”** 的按钮。点击它，从你的电脑里选一个本地的 PDF 或 Excel 文件。
     2.   在 `file` 下面的新行里，继续添加其他文本参数。它们的类型保持默认的 **`Text`** 即可

---

## 六、架构设计・通信策略（轮询 VS 推送）

### 1. 轮询 CPU 占用高，为何不采用后端主动反馈（接收请求返回解析中，数据解析入库完成后再返回成功状态）？

1.   前端发一次请求，后端只能给**一次**响应。后端把数据返回给前端之后，这条 HTTP 连接就在逻辑上**结束并断开**了。后端无法在几秒钟或几分钟后，顺着这条已经结束的连接，再主动给前端发一个“解析成功”。

2.   如果让后端收到文件后，不立刻返回，直到代码处理完毕之后再返回，这就是传统的**同步处理方案**。

     同步处理的弊端：

     1.   **浏览器超时**：代码执行时间过长，导致后端一直不响应。一般的浏览器或 Nginx 网关在等待超过 30 秒或 60 秒没有收到后端的响应时，会直接报 `504 Gateway Timeout`，前端页面会直接崩溃。
     2.   **线程池耗尽**：在 Spring Boot (Tomcat) 中，每一个 HTTP 请求都会占用一个工作线程。如果同时有 50 个用户在传大文件，Tomcat 的核心线程池瞬间就被占满卡死了，导致整个系统无法处理其他任何用户的正常请求。

     **所以我们必须采用“异步方案”**：接收到文件后，立刻把线程释放掉，先给前端一个“收据（fileId）”，然后后台自己慢慢干活。



### 2. 为什么选择使用轮询的方式来查看任务进度而不是后端主动推送？

在异步方案中，前端要怎么知道后台干完活了呢？通常有两种做法：**前端轮询** 和 **后端主动推送 (WebSocket / SSE)**。

之所以采用**前端轮询**，是基于以下几个现实考量：

1. **性能损耗微乎其微**：因为前端是拿着 `fileId` 去查数据库，`fileId` 是主键（自带聚簇索引）。在 MySQL 中，通过主键查询一条记录的状态，耗时通常在 **0.1 毫秒**级别。就算前端每隔 3 秒查一次，对服务器的 CPU 消耗几乎等于 0。

2. **网络环境的健壮性**：这是一个离线/物理隔离的系统。在这种内网环境下，可能存在各种复杂的防火墙或代理策略。WebSocket 这种长连接技术，一旦网络稍微抖动就会断开，还需要前端写复杂的重连逻辑。而轮询是最原始、最抗造的 HTTP 短连接，容错率极高。

3. **开发成本极低**：写一个查数据库状态的接口只需要 5 行代码。
























