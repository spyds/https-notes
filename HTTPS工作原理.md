# HTTPS的工作原理

## 1.HTTPS与HTTP的区别

### (1) HTTP

**HTTP协议**（超文本传输协议）是 一种基于TCP/IP通信协议来传递数据 （属于应用层）， 主要规定了服务器端与客户端传输的数据格式。

### (2) HTTPS

由于HTTP是以明文的形式进行传输数据，且其在传输过程并没有进行任何的身份确认，因此HTTP是不安全的。

为此在HTTP协议的基础之上，又制定了**HTTPS协议**(安全超文本传输协议)。**HTTPS协议**与**HTTP协议**的关系如下图，其在应用层与数据传输层之间，又加了一层**SSL**（安全传输层），利用SSL保证HTTP传输的安全。

![1618903762607](C:\Users\10560\Desktop\HTTPS工作原理\HTTPS工作原理.assets\1618903762607.png)

## 2.HTTPS的工作流程

HTTPS如何保证数据传输的安全的呢？这就需要了解其HTTPS的工作原理了：

<img src="C:\Users\10560\Desktop\HTTPS工作原理\HTTPS工作原理.assets\1619019887245.png" alt="1619019887245" style="zoom: 80%;" />

HTTPS数据传输的整个过程大致上可以分成两个部分(个人理解) :  SSL握手部分 和 数据传输部分 

**（1）SSL握手部分**

这部分的工作主要由两个目的：1.验证服务器身份(这里主要是用到SSL证书)   2.获取后面用于数据传输加密用的密钥（数据传输时采用的对称加密算法）。

**1.验证服务器身份**

**step1**：客户端发起client hello（SSL握手请求）

| Client Helle包含的信息              | 用处                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| 客户端支持的SSL/TLS版本             | 声明支持SSL/TLS的协议                                        |
| 客户端支持的加密套件(Cipher Suites) | 用于告诉服务器端SSL握手可以采用哪种加密算法：例如：**TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA256**，**基本的形式**是“密钥交换算法-服务身份验证算法-对称加密算法-握手校验算法”  ，**其实际意义**：握手过程中，证书签名使用的RSA算法，如果证书验证正确，再使用ECDHE算法进行密钥交换，握手后的通信使用的是AES256的对称算法分组模式是GCM。验证证书签名合法性使用SHA256作哈希算法检验。 |
| 会话Id`session id`                  | 有值的话，服务器端会复用对应的握手信息，避免短时间内重复握手 |
| 随机数`client-random`               | 用于组成数据传输 对称加密的密钥的一部分                      |



**step2**：服务器端在收到这个`Client Hello`，从中选择服务器支持的版本和套件，发送`Server Hello`消息 .

| Server Hello                                                 | 用处                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 服务器所能支持的最高SSL/TLS版本                              |                                                              |
| 服务器选择的加密套件                                         |                                                              |
| 随机数`server-random`                                        | 用于组成数据传输 对称加密的密钥的一部分                      |
| 会话Id`session id`(用于下次复用当前握手的信息，避免短时间内重复握手。) | 有值的话，服务器端会复用对应的握手信息，避免短时间内重复握手 |
| SSL安全证书（含服务器公钥）                                  | 1.SSL证书用于验证服务器身份； 2.服务器公钥用于加密数据传输的对称密钥 |

**step3**：客户端接受到Server Hello后，会对CA签发的SSL证书验证

 浏览器首先用哈希函数对明文信息的摘要做哈希得到一个哈希值（用到的就是证书中的签名哈希算法SHA256），然后用根CA的公钥对根证书的签名作解密得到另一个哈希值（用到的算法就是RSA非对称算法），如果两个哈希值相等则说明证书没有被篡改过。当然还需校验证书中服务器名称是否合法以及验证证书是否过期. 

​	浏览器会从CA根证书开始校验，直到证书链校验完成，如果整个校验链有一个证书校验失败，整个服务器身份校验失败。

![1619021282922](C:\Users\10560\Desktop\HTTPS工作原理\HTTPS工作原理.assets\1619021282922.png)

**2.产生数据传输加密的密钥**

**step1：**客户端验证网站身份后，预先产生一个随机数`pre-master`，然后利用  `server-random` + `client-random` + `pre-master` 计算出用于数据传输的对称密钥 **master secret**。

**step2：**客户端利用服务器发来的**非对称密钥的公钥**加密`pre-master`发给服务器端，服务器端利用**非对称密钥的私钥**解密拿到`pre-master`。这样服务器通过 `server-random` + `client-random` + `pre-master` 也拿到了**master secret**



**（2）数据传输部分**

这一部分简单了，客户端利用**master secret**对称加密数据，然后服务端利用**master secret**对称解密得到数据。



## 3.SSL证书

 		SSL证书是由CA机构颁发的，是一个HTTPS网站的身份证明，SSL证书里面包含了网站的域名，证书有效期，证书的颁发机构以及用于加密传输密码的公钥等信息。

<img src="C:\Users\10560\Desktop\HTTPS工作原理\HTTPS工作原理.assets\1619024470387.png" alt="1619024470387" style="zoom:67%;" />



## 4.如何自己生成SSL证书

### （1）简介

可以利用JDK自带的Keytool数据证书管理工具（在jdk的bin目录下）生成自己的SSL证书。

 Keytool会生成一个keystore的文件， 其主要包含两种数据：

（1） 密钥实体（Key entity）—— 非对称加密的私钥和公钥

（2）可信任的证书实体（trusted certificate entries）——证书 & 公钥



###  **（2）keytool 常用命令**

| 命令         | 作用                                                         |
| ------------ | ------------------------------------------------------------ |
| -genkey      | 创建.keystore文件                                            |
| -alias       | key别名                                                      |
| -keystore    | 指定密钥库的名称(产生的各类信息将不在.keystore文件中)        |
| -keyalg      | 指定密钥的算法 (如 RSA  DSA（如果不指定默认采用DSA）)        |
| -validity    | 指定创建的证书有效期多少天                                   |
| -keysize     | 指定密钥长度                                                 |
| -storepass   | 指定密钥库的密码(获取keystore信息所需的密码)                 |
| -keypass     | 指定别名条目的密码(私钥的密码)                               |
| -dname       | 指定证书拥有者信息 例如：  "CN=名字与姓氏,OU=组织单位名称,O=组织名称,L=城市或区域名称,ST=州或省份名称,C=单位的两字母国家代码" |
| -list        | 显示密钥库中的证书信息      keytool -list -v -keystore 指定keystore -storepass 密码 |
| -v           | 显示密钥库中的证书详细信息                                   |
| -export      | 将别名指定的证书导出到文件  keytool -export -alias 需要导出的别名 -keystore 指定keystore -file 指定导出的证书位置及证书名称 -storepass 密码 |
| -file        | 参数指定导出到文件的文件名                                   |
| -delete      | 删除密钥库中某条目    keytool -delete -alias 指定需删除的别名  -keystore 指定keystore  -storepass 密码 |
| -printcert   | 查看导出的证书信息          keytool -printcert -file 证书名.crt |
| -storepasswd | 修改keystore口令      keytool -storepasswd -keystore e:/yushan.keystore(需修改口令的keystore) -storepass 123456(原始密码) -new yushan(新密码) |
| -import      | 将已签名数字证书导入密钥库: keytool -import -alias 指定导入条目的别名 -keystore 指定keystore -file 需导入的证书 |



### **（3）示例**

**1.创建证书**

```bash
 keytool -genkey -alias store_name[别名] -keypass password[别名密码] -keyalg RSA[算法] -keysize 1024[密钥长度] -validity 365[有效期，天单位] -keystore d:/store_name.keystore[指定生成证书的位置和证书名称] -storepass 123456[获取keystore信息的密码]
```

```bash
keytool -genkey -alias mykey -keypass mykey_password -keyalg RSA -keysize 1024 -validity 365 -keystore  d:/keystore/mykeystore.keystore -storepass mykeystore_password -dname "CN=(名字与姓氏), OU=(组织单位名称), O=(组织名称), L=(城市或区域名称), ST=(州或省份名称), C=(单位的两字母国家代码)"

keytool -genkey -alias mykey -keypass mykey_password -keyalg RSA -keysize 1024 -validity 365 -keystore  d:/keystore/mykeystore.keystore -storepass mykeystore_password -dname "CN=user_name, OU=company_name, O=org_name, L=hangzhou, ST=zhengjiang, C=cn"
```



**2.查看证书**

```bash
 keytool -list  -v -keystore d:/keystore/mykeystore.keystore -storepass mykeystore_password
```

![1619101557857](C:\Users\10560\Desktop\HTTPS工作原理\HTTPS工作原理.assets\1619101557857.png)



**3.导出证书**

```bash
keytool -export -alias mykey -keystore d:/keystore/mykeystore.keystore -file d:/keystore/mykey.crt -storepass mykeystore_password
```

![1619101880342](C:\Users\10560\Desktop\HTTPS工作原理\HTTPS工作原理.assets\1619101880342.png)



**4.导入证书**

```bash
keytool -import -alias other_mykey -file e:/other_mykey.crt -keystore d:/keystore/mykeystore.keystore -storepass mykeystore_password 
```



**5、证书条目的删除**

```bash
 keytool -delete -alias other_mykey -keystore d:/keystore/mykeystore.keystore -storepass mykeystore_password
```



**6、证书条目口令的修改：** 

```bash
keytool -keypasswd -alias mykey -keypass mykey_password -new new_mykey_password -keystore d:/keystore/mykeystore.keystore -storepass mykeystore_password
```



**7、keystore口令的修改：** 

```bash
keytool -storepasswd -keystore d:/keystore/mykeystore.keystore -storepass mykeystore_password -new new_mykeystore_password
```



**8、修改keystore中别名为mykey的信息**

```bash
keytool -selfcert -alias mykey -keypass mykey_password -keystore d:/keystore/mykeystore.keystore -storepass mykeystore_password -dname "cn=new_user_name,ou=new_company_name,o=new_org_name,c=us"
```

