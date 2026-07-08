# Web 安全基础

> Web 安全是互联网系统的底线。面试中安全问题的回答质量直接体现候选人的工程成熟度。本篇覆盖 XSS、SQL 注入、CSRF、SSRF 四大经典攻击及防御体系，配合安全设计原则。

---

## 目录

- [1. 安全设计原则](#1-安全设计原则)
- [2. XSS（跨站脚本攻击）](#2-xss跨站脚本攻击)
- [3. SQL 注入](#3-sql-注入)
- [4. CSRF（跨站请求伪造）](#4-csrf跨站请求伪造)
- [5. SSRF（服务器端请求伪造）](#5-ssrf服务器端请求伪造)
- [6. 其他常见攻击](#6-其他常见攻击)
- [7. 安全头部与防御体系](#7-安全头部与防御体系)
- [8. 面试追问应对](#8-面试追问应对)

---

## 1. 安全设计原则

```
纵深防御（Defense in Depth）：
  不依赖单一防线，多层防护叠加
  例：输入校验 + 参数化查询 + WAF + 最小权限

最小权限原则（Least Privilege）：
  每个组件、每个用户只拥有完成任务的最小权限
  例：数据库账号只用读写权限，不用 DBA 权限
  例：应用服务不暴露 22 端口，不运行 root

不信任任何输入（Never Trust Input）：
  所有外部输入（用户输入、第三方回调、文件上传）都视为不可信
  必须校验、清洗、编码后才能使用

安全默认值（Secure by Default）：
  默认开启安全选项，默认关闭危险功能
  例：Cookie 默认 HttpOnly + Secure + SameSite
  例：接口默认需要鉴权，白名单开放特定接口

失败安全（Fail Secure）：
  系统异常时，默认拒绝访问（而非默认允许）
  例：鉴权服务挂了，所有请求拒绝（而非放行）
  例：权限检查出错，拒绝操作（而非默认允许）
```

---

## 2. XSS（跨站脚本攻击）

### 2.1 攻击原理

```
XSS：攻击者将恶意脚本注入网页，用户浏览器执行恶意脚本

三种类型：

  反射型（Reflected XSS）：
    攻击步骤：
      1. 攻击者构造含恶意脚本的 URL（如 https://site.com/search?q=<script>alert(1)</script>）
      2. 诱导用户点击链接（钓鱼邮件、论坛帖子）
      3. 服务器将搜索词原样返回页面（未转义）
      4. 用户浏览器执行 <script>，窃取 Cookie 或伪造请求
    特点：恶意脚本在 URL 中，不持久化，需要用户点击

  存储型（Stored XSS）：
    攻击步骤：
      1. 攻击者在评论区提交 <script>alert(document.cookie)</script>
      2. 服务器保存到数据库（未转义）
      3. 其他用户浏览该页面，浏览器执行恶意脚本
    特点：恶意脚本持久化到数据库，危害最大，所有浏览用户都受影响

  DOM 型（DOM-based XSS）：
    攻击步骤：
      1. 攻击者构造 URL：https://site.com/#data=<img src=x onerror=alert(1)>
      2. 前端 JavaScript 从 URL hash 读取 data，通过 innerHTML 写入 DOM
      3. 浏览器解析执行恶意脚本
    特点：恶意脚本在客户端生成，服务器不返回恶意内容（绕过服务端过滤）
```

### 2.2 防御方案

```
核心原则：永远不要信任用户输入，输出时必须编码

1. 输入过滤（第一道防线，但不可靠）：
   过滤 <script> 标签等危险字符
   但攻击者可绕过（如 <scr<script>ipt>、编码绕过）
   不能单独依赖输入过滤

2. 输出编码（核心防御）：
   HTML 上下文：将 < 转义为 &lt;，> 转义为 &gt;，" 转义为 &quot;
   JavaScript 上下文：使用 JSON.stringify，对特殊字符编码
   URL 上下文：使用 encodeURIComponent
   CSS 上下文：严格校验，只允许白名单字符

   推荐：使用框架自动转义（如 React 的 JSX 默认转义，Thymeleaf 的 th:text）

3. Content Security Policy（CSP）：
   通过 HTTP 响应头限制页面可执行脚本的来源
   Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com
   效果：即使 XSS 注入成功，外部脚本无法执行，inline script 被阻止

4. Cookie 安全：
   HttpOnly：禁止 JavaScript 读取 Cookie（document.cookie 返回空）
   Secure：只允许 HTTPS 传输 Cookie
   SameSite：防止 Cookie 随跨站请求发送（Strict / Lax / None）

5. 其他措施：
   对富文本编辑器（如 Markdown、富文本），使用白名单过滤（DOMPurify）
   验证码 / WAF：防止自动化 XSS 攻击
```

### 2.3 面试要点

```
"XSS 三种类型的区别"：
  反射型：URL 中，不持久化，需诱导点击
  存储型：存入数据库，持久化，所有浏览者都受害
  DOM 型：纯前端漏洞，服务器不返回恶意内容，服务端过滤无效

"React 为什么能防御 XSS"：
  JSX 默认将插入的内容作为文本节点处理，自动转义 HTML 标签
  但 dangerouslySetInnerHTML 会绕过保护，需确保内容安全

"CSP 是什么，怎么配置"：
  Content Security Policy，通过 HTTP 头限制页面可加载/执行的资源来源
  核心指令：default-src（默认）、script-src（脚本来源）、style-src（样式来源）
  推荐生产环境至少配置：default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'
```

---

## 3. SQL 注入

### 3.1 攻击原理

```
SQL 注入：攻击者通过输入篡改 SQL 语句，执行非预期操作

攻击示例：
  登录接口：SELECT * FROM users WHERE username = '$username' AND password = '$password'
  攻击输入：username = admin' OR '1'='1'; --
  拼接后：SELECT * FROM users WHERE username = 'admin' OR '1'='1'; --' AND password = 'xxx'
  结果：WHERE 条件永远为真，绕过密码验证

  删库示例：
  输入：'; DROP TABLE users; --
  拼接后：SELECT * FROM users WHERE username = ''; DROP TABLE users; --'
  结果：执行了删除表操作

危害等级：
  读取任意数据（拖库）
  修改/删除数据
  执行系统命令（如 xp_cmdshell，取决于数据库权限）
  绕过认证（无需密码登录）
```

### 3.2 防御方案

```
核心防御：参数化查询（Prepared Statement）

1. 参数化查询（根本解决方案）：
   将 SQL 语句和数据分离，数据作为参数传入，数据库预编译后执行
   SELECT * FROM users WHERE username = ? AND password = ?
   参数：['admin', '123456']
   数据库将参数视为纯数据，不会作为 SQL 语法解析

   注意：参数化查询必须参数化所有动态部分，不能拼接表名、列名（这些用白名单）

2. ORM 框架：
   MyBatis 的 #{} 自动参数化
   JPA 的 JPQL 自动参数化
   注意：MyBatis 的 ${} 是字符串拼接，有注入风险，只用于非用户输入（如 ORDER BY 列名）

3. 输入校验（辅助）：
   白名单校验：如用户名只允许字母数字，长度限制
   但仅依赖输入校验不够（业务上可能允许特殊字符）

4. 最小权限：
   应用数据库账号只用读写权限，禁止 DROP、TRUNCATE 等高危操作
   即使注入成功，权限受限，危害降低

5. WAF / SQL 防火墙：
   拦截常见 SQL 注入特征（如 UNION SELECT、sleep()、xp_cmdshell）
   作为最后一道防线，不能替代参数化查询

错误做法：
  黑名单过滤（如 replace ' 为 ''）：攻击者可绕过编码（如 Unicode 编码）
  前端校验：前端校验可被绕过（直接请求 API）
```

### 3.3 面试要点

```
"MyBatis #{} 和 ${} 的区别"：
  #{} 是预编译参数（PreparedStatement），防 SQL 注入
  ${} 是字符串拼接，有 SQL 注入风险，只用于非用户输入场景（如动态表名、列名）

"参数化查询的原理"：
  SQL 语句先发送到数据库预编译（生成执行计划），参数随后作为纯数据传入
  数据库不会将参数解析为 SQL 语法，因此无法注入

"ORM 能完全防止 SQL 注入吗"：
  正常使用能，但特殊场景可能绕过：
  1. 使用原生 SQL 且拼接参数
  2. 动态 ORDER BY 使用 ${} 且未白名单校验
  3. 存储过程内部拼接 SQL
```

---

## 4. CSRF（跨站请求伪造）

### 4.1 攻击原理

```
CSRF：攻击者诱导用户在已登录状态下，向目标网站发起非预期的请求

攻击场景：
  1. 用户登录银行网站 bank.com，Cookie 保存在浏览器
  2. 用户未退出，访问攻击者网站 evil.com
  3. evil.com 的页面包含：<img src="https://bank.com/transfer?to=attacker&amount=10000">
  4. 浏览器自动携带 bank.com 的 Cookie 发送请求
  5. 银行服务器验证 Cookie 通过，执行转账

关键点：
  - 用户已登录目标网站（Cookie 有效）
  - 攻击者构造请求（GET/POST）
  - 浏览器自动携带目标域 Cookie（同源策略不限制跨站请求发送，只限制读取响应）
  - 服务器只验证 Cookie，不验证请求来源

与 XSS 的区别：
  XSS：利用用户对网站的信任，执行恶意脚本
  CSRF：利用网站对浏览器的信任，浏览器自动携带 Cookie
```

### 4.2 防御方案

```
1. CSRF Token（最可靠）：
   服务器生成随机 Token，嵌入表单或页面 meta
   表单提交时携带 Token，服务器验证 Token 有效性
   攻击者无法获取 Token（同源策略限制读取），无法伪造请求

   实现方式：
   - 同步 Token 模式：页面渲染时注入 <input type="hidden" name="csrf_token" value="xxx">
   - 双重 Cookie 模式：Cookie 中存 Token，请求头或参数中也带 Token，服务器比对
   - 自定义请求头（如 X-CSRF-Token）：非简单请求，跨域时浏览器会预检（CORS preflight），攻击者无法伪造

2. SameSite Cookie（现代浏览器支持）：
   SameSite=Strict：完全禁止跨站发送 Cookie（最严格）
   SameSite=Lax：允许 top-level 导航（如链接跳转），禁止 POST / iframe 请求携带 Cookie（推荐）
   SameSite=None：允许跨站发送，但必须配合 Secure（HTTPS）

   注意：SameSite 是浏览器行为，旧浏览器不支持（需配合 Token）

3. 验证 Referer / Origin：
   检查请求头的 Referer 或 Origin，只允许来自本站
   但 Referer 可被禁用或篡改，不可靠，作为辅助手段

4. 关键操作二次验证：
   转账、修改密码等敏感操作，要求输入密码、短信验证码
   即使 CSRF 成功，也无法完成二次验证

5. 自定义请求头 + CORS：
   前端发送请求时带自定义头（如 X-Requested-With: XMLHttpRequest）
   跨域简单请求无法携带自定义头，攻击者无法伪造
   但需确保 CORS 配置正确，不开放 * 给敏感接口
```

### 4.3 面试要点

```
"CSRF 和 XSS 的区别"：
  XSS 是攻击者利用网站漏洞，在用户浏览器执行恶意脚本，窃取信息或冒充用户操作
  CSRF 是攻击者利用用户已登录的状态，诱导浏览器自动发送请求，网站误以为是用户主动操作
  两者可结合：XSS 获取 Token 后，用 CSRF 发起请求

"为什么 SameSite=Lax 不能防御所有 CSRF"：
  SameSite=Lax 允许 GET 请求跨站携带 Cookie（如通过链接跳转）
  如果敏感操作使用 GET（本身就不对），或攻击者利用 top-level 导航，仍可能攻击
  而且旧浏览器不支持 SameSite，需要 Token 作为 fallback

"CSRF Token 放在哪里，怎么验证"：
  放在表单 hidden 字段、HTTP 头（X-CSRF-Token）、或 Cookie 中（双重 Cookie 模式）
  验证：服务器生成 Token 存 Session / Redis，请求时比对，正确且未过期则通过
```

---

## 5. SSRF（服务器端请求伪造）

### 5.1 攻击原理

```
SSRF：攻击者诱导服务器向攻击者指定的地址发起请求，访问内网资源或外部系统

典型场景：
  1. 图片下载/上传：用户输入图片 URL，服务器下载后展示
     攻击输入：http://169.254.169.254/latest/meta-data/（AWS 元数据服务，获取临时凭证）
     服务器发起请求，泄露云厂商临时凭证

  2. 网页抓取/URL 预览：输入 URL，服务器抓取内容返回
     攻击输入：http://localhost:8080/admin/ 或 http://127.0.0.1:3306（扫描内网端口）

  3. Webhook / 回调 URL：用户配置回调地址，服务器发送 HTTP 请求
     攻击输入：file:///etc/passwd（读取服务器本地文件）

危害：
  访问内网服务（如 Redis、MySQL、K8s API、云厂商元数据服务）
  读取服务器本地文件（file:// 协议）
  内网端口扫描（探测内网服务存活）
  绕过防火墙（通过服务器做跳板）
  云环境：获取临时凭证，接管云资源
```

### 5.2 防御方案

```
1. 白名单限制 URL（最有效）：
   只允许访问指定的域名/IP（如 https://cdn.example.com）
   禁止访问内网 IP（10.0.0.0/8、172.16.0.0/12、192.168.0.0/16、127.0.0.1）
   禁止非 HTTP 协议（file://、gopher://、ftp://）

2. 解析后校验（绕过 DNS 重绑定）：
   先解析域名获取 IP，再校验 IP 是否在白名单
   注意：需禁用 302 跳转（或跳转后再次校验），防止重定向到内网

3. 禁用危险协议：
   只允许 http / https，禁用 file、ftp、gopher、dict 等协议
   禁用 302 跳转（或跳转后重新校验目标）

4. 网络隔离：
   服务器部署在 DMZ 区，无法访问内网核心服务
   使用独立的下载服务（沙箱环境），与主应用隔离

5. 云厂商防护：
   AWS：IMDSv2（元数据服务需要 Token，防止 SSRF）
   阿里云：RAM 角色绑定，元数据访问需认证

面试要点：
  "SSRF 和 CSRF 的区别"：
  CSRF 是利用用户浏览器自动发送请求，攻击目标网站
  SSRF 是利用服务器发起请求，攻击内网或外部系统（服务器作为跳板）

  "SSRF 怎么读本地文件"：使用 file:// 协议，如 file:///etc/passwd
  "怎么防御 SSRF"：白名单域名、校验 IP 非内网、禁用危险协议、网络隔离
```

---

## 6. 其他常见攻击

### 6.1 点击劫持（Clickjacking）

```
攻击：攻击者用 iframe 嵌套目标页面，用透明层覆盖，诱导用户点击
防御：X-Frame-Options: DENY / SAMEORIGIN（禁止被 iframe 嵌套）
      CSP: frame-ancestors 'none'（现代替代方案）
```

### 6.2 会话固定（Session Fixation）

```
攻击：攻击者预先设置 Session ID，诱导用户使用该 Session ID 登录
      登录后攻击者用相同 Session ID 冒充用户
防御：登录后重新生成 Session ID
      Cookie 设置 HttpOnly + Secure + SameSite
```

### 6.3 不安全的反序列化

```
攻击：攻击者提交恶意序列化数据，服务端反序列化时执行任意代码
      Java：RCE（远程代码执行）
      PHP：phar 反序列化漏洞
防御：不反序列化不可信数据
      使用 JSON 替代 Java 原生序列化
      反序列化前校验类型（白名单）
```

### 6.4 敏感信息泄露

```
场景：API 返回完整错误堆栈（泄露代码路径、库版本）
      日志打印密码、Token
      前端代码包含 API Key、数据库密码
      .git 目录暴露
防御：生产环境关闭详细错误信息
      日志脱敏（身份证号、手机号、密码打码）
      配置中心管理密钥，不硬编码
      禁止访问 .git、.env、WEB-INF 等敏感目录
```

---

## 7. 安全头部与防御体系

### 7.1 安全响应头

```
HTTP 安全头部（浏览器级别防护）：

  Strict-Transport-Security (HSTS)：
    强制浏览器只用 HTTPS 访问，防止 SSL 剥离攻击
    max-age=31536000; includeSubDomains

  Content-Security-Policy (CSP)：
    限制页面可加载的资源来源，防止 XSS 执行恶意脚本
    default-src 'self'; script-src 'self' https://cdn.example.com; img-src 'self' data:

  X-Content-Type-Options：
    nosniff：禁止浏览器猜测 MIME 类型（防止上传图片实际是 JS，被当作脚本执行）

  X-Frame-Options：
    DENY / SAMEORIGIN：防止点击劫持

  Referrer-Policy：
    strict-origin-when-cross-origin：跨域时只发送 origin，不发送完整路径（防止 URL 参数泄露）

  Permissions-Policy：
    限制浏览器 API 权限（如禁止摄像头、麦克风、地理位置）
```

### 7.2 防御体系层次

```
防御层次（从外到内）：

  边界层：
    WAF（Web 应用防火墙）
    CDN（DDoS 防护、Bot 管理）
    网络防火墙（限制端口、IP 白名单）

  应用层：
    输入校验（白名单）
    输出编码（HTML/JS/URL 编码）
    参数化查询（防 SQL 注入）
    CSRF Token（防 CSRF）
    SSRF 白名单（防 SSRF）

  框架层：
    使用安全框架（Spring Security、Shiro）
    自动 XSS 转义（React、Thymeleaf）
    ORM 防注入（MyBatis #{}、JPA）

  数据层：
    最小权限数据库账号
    数据加密（敏感字段加密存储）
    脱敏（日志、接口返回脱敏）

  运维层：
    HTTPS 全链路（TLS 1.2+）
    安全响应头（CSP、HSTS、X-Frame-Options）
    漏洞扫描（依赖库漏洞扫描、渗透测试）
    日志审计（操作日志、访问日志）
```

---

## 8. 面试追问应对

**Q1：XSS 的防御方案有哪些？**
> 核心是输出编码，不能仅依赖输入过滤。具体方案：1）输出编码（HTML 上下文转义 < > " 等，JS 上下文用 JSON.stringify）；2）CSP（Content Security Policy）限制脚本来源；3）Cookie 设置 HttpOnly（防 JS 读取）、Secure（HTTPS 传输）、SameSite（防跨站）；4）使用框架自动转义（React JSX 默认转义）；5）富文本用白名单过滤（DOMPurify）。

**Q2：SQL 注入怎么防御？为什么参数化查询能防注入？**
> 根本方案是参数化查询（Prepared Statement）。原理：SQL 语句和数据分离，语句先发送到数据库预编译，参数随后作为纯数据传入，数据库不会将参数解析为 SQL 语法，因此无法注入。其他辅助措施：ORM 框架（MyBatis #{}）、输入白名单校验、最小权限数据库账号、WAF。注意 MyBatis 的 ${} 是字符串拼接，有注入风险，只用于非用户输入场景。

**Q3：CSRF 的 Token 方案具体怎么实现？**
> 服务器生成随机 Token 存入 Session 或 Cookie，页面渲染时嵌入表单 hidden 字段或 HTTP 头（X-CSRF-Token）。表单提交时携带 Token，服务器验证请求中的 Token 与存储的 Token 是否一致。攻击者无法获取 Token（同源策略限制跨域读取），因此无法伪造请求。现代方案可配合 SameSite=Lax Cookie，但旧浏览器不支持，Token 作为 fallback。

**Q4：SSRF 有什么危害？怎么防御？**
> 危害：服务器作为跳板访问内网（如 Redis、MySQL、K8s API）、读取本地文件（file://）、获取云厂商临时凭证（如 AWS 169.254.169.254）。防御：1）白名单限制 URL 域名；2）解析域名后校验 IP 是否在内网段；3）禁用非 HTTP 协议（file、ftp、gopher）和 302 跳转；4）网络隔离（服务器部署在 DMZ，无法访问内网）；5）云厂商启用 IMDSv2（元数据访问需 Token）。

**Q5：Cookie 的 SameSite 属性有什么用？**
> SameSite 控制 Cookie 是否随跨站请求发送。Strict：完全禁止跨站发送（最安全，但用户体验差，如从外部链接进入需重新登录）。Lax：允许 top-level 导航（如链接跳转），禁止 POST/iframe/异步请求携带 Cookie（推荐）。None：允许跨站发送，但必须配合 Secure（HTTPS）。SameSite 是 CSRF 防御的重要补充，但旧浏览器不支持，不能完全替代 Token。

**Q6：生产环境怎么发现安全漏洞？**
> 1）依赖扫描（Snyk、OWASP Dependency Check）发现第三方库漏洞；2）代码审计（SonarQube、CodeQL）发现代码级安全问题；3）渗透测试（内部或外部团队模拟攻击）；4）WAF 日志分析异常请求；5）Bug Bounty 或 SRC（安全响应中心）收集外部报告；6）自动化漏洞扫描（Nessus、AWVS）。

**Q7：安全头部有哪些，分别作用是什么？**
> HSTS（Strict-Transport-Security）强制 HTTPS；CSP（Content-Security-Policy）限制资源来源防 XSS；X-Content-Type-Options: nosniff 禁止 MIME 嗅探；X-Frame-Options 防点击劫持；Referrer-Policy 控制 Referrer 信息泄露；Permissions-Policy 限制浏览器 API 权限。

---

## 关联阅读

- [04-业务系统实战/00-电商交易核心系统设计](../04-业务系统实战/00-电商交易核心系统设计.md) — 电商系统的权限与安全设计
- [03-分布式架构与设计/03-Spring Cloud 生态](../03-分布式架构与设计/03-Spring%20Cloud%20生态.md) — 网关层鉴权与安全
- [05-稳定性与可观测/00-限流与熔断策略体系](../05-稳定性与可观测/00-限流与熔断策略体系.md) — 安全与稳定性的结合

---

> **一句话总结**：安全 = 不信任输入 + 输出编码 + 最小权限 + 纵深防御。XSS 防输出编码，SQL 注入防参数化查询，CSRF 防 Token + SameSite，SSRF 防白名单 + 网络隔离。安全头部是浏览器级防护，WAF 是边界防护，两者都不能替代代码层安全。
