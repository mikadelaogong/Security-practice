# SQL注入-联合注入漏洞复现（sqli-labs Less-1）

## 一、漏洞简介
SQL注入是指攻击者通过在输入框中插入恶意SQL代码，欺骗数据库执行非授权查询的漏洞。
联合注入利用 UNION SELECT 语句将查询结果合并回显到页面，适用于页面有数据回显的场景。

**影响版本**：PHP + MySQL（未使用参数化查询的场景）

**漏洞危害**：数据库敏感信息泄露、数据篡改、甚至获取服务器控制权


## 二、实验环境

| 组件 | 版本/说明 |
|:---|:---|
| 集成环境 | PHPStudy（Apache + MySQL） |
| 靶场 | sqli-labs（Less-1） |
| 浏览器 | Chrome / Firefox |


## 三、漏洞复现步骤

### 3.1 寻找注入点

访问 Less-1：### 3.1 寻找注入点
`http://127.0.0.1/sqli-labs/Less-1/?id=1`，页面正常回显。

在参数后添加单引号，页面返回数据库报错，确认存在SQL注入。

![注入点报错](image.png)


### 3.2 判断字段数
使用 ORDER BY 判断字段数，`order by 4` 报错，说明共3个字段。

![判断字段数](b9fdde05ba970f2896c6fe0728ce612a.png)


### 3.3 确定回显位
`?id=-1' union select 1,2,3--+`，页面回显数字2和3。

![爆回显位](8dadbf5b32eb29c4b8a81afbeb6f9350.png)


### 3.4 获取数据库名
`?id=-1' union select 1,2,database()--+`，回显`security`。

![获取数据库名](3d72d34633d8cff4e72e8f3d8efc8b5e.png)


### 3.5 获取所有表名
回显表名列表：`emails,referers,uagents,users`

![获取表名](b19101d7f98bbea0d67908ae97203e7d.png)


### 3.6 获取列名
回显列名：`id,username,password`

![获取列名](453a3677009561b415a6381aa43a356f.png)


### 3.7 获取数据
成功获取所有用户名和密码。

![获取数据](step7.png)
