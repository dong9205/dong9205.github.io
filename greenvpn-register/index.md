# GreenVPN注册


# GreenVPN注册 {#greenvpn-register}

# 一、greenvpn下载

> 官网地址: [GreenVPN - Free VPN Service | Free VPN Software](https://www.greenvpn.app/)
> 

# 二、注册greenvpn新账户

## 2.1 获取临时邮箱

> 临时邮箱获取地址：[更改电子邮件地址 - Temp Mail (temp-mail.org)](https://temp-mail.org/zh/change)
> 

打开页面后，复制邮件，然后点击刷新（会等待新邮件的发送）

<!-- ![Untitled](GreenVPN%E6%B3%A8%E5%86%8C%2039d34b4ab9db4a438d32867a1202867a/Untitled.png)
 -->
{{< figure src="GreenVPN%E6%B3%A8%E5%86%8C%2039d34b4ab9db4a438d32867a1202867a/Untitled.png" title="" >}}

## 2.2 注册greenvpn

### 通过web页面注册

**打开地址: [GreenVPN - Free VPN Service | Free VPN Software](https://www.greenvpn.app/login.shtml)输入邮件注册即可**

<!-- ![Untitled](GreenVPN%E6%B3%A8%E5%86%8C%2039d34b4ab9db4a438d32867a1202867a/Untitled%201.png) -->
{{< figure src="GreenVPN%E6%B3%A8%E5%86%8C%2039d34b4ab9db4a438d32867a1202867a/Untitled%201.png" title="" >}}

### 通过greenvpn客户端注册

安装后点击**用户登录/注册账户**，然后点击**注册账户**即可

<!-- ![Untitled](GreenVPN%E6%B3%A8%E5%86%8C%2039d34b4ab9db4a438d32867a1202867a/Untitled%202.png) -->
{{< figure src="GreenVPN%E6%B3%A8%E5%86%8C%2039d34b4ab9db4a438d32867a1202867a/Untitled%202.png" title="" >}}

### 使用powershell终端命令注册

**①打开终端**

1）点击桌面下方的Windows按钮或者俺键盘的Win键

<!-- ![Untitled](GreenVPN%E6%B3%A8%E5%86%8C%2039d34b4ab9db4a438d32867a1202867a/Untitled%203.png) -->
{{< figure src="GreenVPN%E6%B3%A8%E5%86%8C%2039d34b4ab9db4a438d32867a1202867a/Untitled%203.png" title="" >}}

2）搜索powershell打开

<!-- ![Untitled](GreenVPN%E6%B3%A8%E5%86%8C%2039d34b4ab9db4a438d32867a1202867a/Untitled%204.png) -->
{{< figure src="GreenVPN%E6%B3%A8%E5%86%8C%2039d34b4ab9db4a438d32867a1202867a/Untitled%204.png" title="" >}}

**②注册账户**

```powershell
curl.exe -H "User-Agent:Mozilla/5.0" -H "Content-Type:application/x-www-form-urlencoded" -X POST "https://www.wzjsq.xyz/regist.shtml?target=&userName=jasadi1014@ibtrades.com&password1=qazwsx123123&password2=qazwsx123123&device=web&identifier=web&register_submit=Sign+up"
```

<!-- ![Untitled](GreenVPN%E6%B3%A8%E5%86%8C%2039d34b4ab9db4a438d32867a1202867a/Untitled%205.png) -->
{{< figure src="GreenVPN%E6%B3%A8%E5%86%8C%2039d34b4ab9db4a438d32867a1202867a/Untitled%205.png" title="" >}}

> **userName=jasadi1014@[ibtrades.com](http://ibtrades.com/)：邮箱地址
password1=qazwsx123123：密码
password2=qazwsx123123：密码**
> 

### 注册1GB流量的账户

默认注册的邮箱是200MB或者500MB, 如果要注册1GB流量的账号，需要使用非中国大陆的IP地址. 所以可以先注册一个正常的账户，然后连接VPN后通过非大陆的IP再次进行账号注册，即可注册1GB流量的账户

① 连接VPN后使用WEB页面注册

② 使用powershell命令行时，增加VPN代理

-x 是指定代理的命令，greenvpn会默认监听8080作为代理端口

```powershell
curl.exe -x 127.0.0.1:8080 -H "User-Agent:Mozilla/5.0" -H "Content-Type:application/x-www-form-urlencoded" -X POST "https://www.wzjsq.xyz/regist.shtml?target=&userName=jasadi1014@ibtrades.com&password1=qazwsx123123&password2=qazwsx123123&device=web&identifier=web&register_submit=Sign+up"
```


### greenvpn的加速类型有"网页/视频", "软件/游戏"

#### 网页/视频

原理是在"internet属性-->连接-->局域网设置"中增加一个代理服务器，
{{< figure src="局域网设置-代理服务器.png" title="局域网设置-代理服务器" >}}

#### 软件/游戏

这个就是一个网卡级别的VPN了，除了浏览器，其他软件访问公网的时候也走VPN

## ApiFox自动化邮件注册

xuan.dong 在 Apifox 邀请你加入项目 temp-mail https://app.apifox.com/invite/project?token=0W6Td6HySKUkqUJPeeesP

{{< figure src="apifox-auto-register.png" title="" >}}

