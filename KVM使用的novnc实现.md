#### 需求软件 ####
1.sqlite3(TokenDB采用sqlite3数据库)

2.python

3.gcc,gcc-c++(numpy依赖)

4.numpy(websockify依赖)

5.websockify(使用自定义版本,使用TokenDB而不是TokenFile,更方便管理token,见该资源包)

_ _ _

#### 安装软件 ####
```
yum install -y sqlite
yum install -y python
yum install -y gcc gcc-c++
```

使用 numpy压缩包(见资源包),解压,安装

```
cd <numpy-root-dir> && python setup.py install
```

将资源包中的 novnc_service.py 拷贝到 /opt/kvm-novnc/ 下,使用 iptables 放开 9000 端口允许
计算节点的CloudKvmAPI的调用:

```
iptables -I INPUT -p tcp --dport 9000 -j ACCEPT
```

进入资源包的 websockify-master目录下,使用命令安装:

```
python setup.py install
```

拷贝资源包中的 novnc web站点 (下面有vnc_auto.html文件那个目录) 到 /opt/kvm-novnc

资源包链接:[点击这里](https://github.com/guojy8993/TinyCloud/tree/master/novnc)

_ _ _

#### 开启服务 ####

```
nohup websockify --web /opt/kvm-novnc/novnc      \
               --token-plugin=TokenDB      \
               --token-source=/opt/kvm-novnc/vnc_tokens.db  6080 >> /opt/kvm-novnc/novnc.log &
nohup /usr/bin/python /opt/kvm-novnc/novnc_service.py &
```

说明:

1.TokenDB 是我们自定义的Token管理类,参考websockify下的token_plugin.py

2.web选项指定novnc web站点的路径

3.token-source指定存放token的数据库文件

4.6080 是默认 webnovnc 端口

5.注意 --token-source 与 novnc_service.py中 `sqlite_db`保持一致

6.如果需要使用自定义 websockify端口 或 novnc_service.py 的监听,注意放开防火墙端口

7.TokenDB负责初始化tokens表,novnc_service.py则仅仅是操作tokens表记录,因此注意服务启动顺序

_ _ _

#### 前端调用 ####

客户业务机公网视图连接 http://kvm.webvnc.com/vnc_auto.html?token=83221476-320a-4a25-a881-7cffa3362583

注意: http://kvm.webvnc.com 解析到对应的vnc服务器的对应端口(e.g:116.255.166.126:6080)

注意: http://kvm.webvnc.com 是此处作为示例的域名,具体情况具体配置

_ _ _

#### 请求处理流程 ####
1.客户点击页面"视图"按钮

2.系统后台触发openVNC

3.触发计算节点(客户机所在宿主)的CloudAPI的openVNC

4.CloudAPI 获取本机ip,以及客户机uuid,vnc端口

5.触发NoVNC服务器的novnc_service: 或生成 token记录或获取尚未过期的token记录(并更新token包含
  的信息,以及到期时间,特别是vnc端口,防止后续的websockify,将vnc视图代理到别的客户机上去) 

6.novnc_service返回token字符串(而不是数据库记录)给CloudAPI,CloudAPI对Jave接口响应

7.Java接口获取token,根据配置文件的novnc链接模版,以及token,拼装出vnc_url(e.g:
  http://kvm.webvnc.com/vnc_auto.html?token=$TOKEN),返回前台

8.前台页面跳转vnc_url链接

9.前台页面的vnc_url请求将token传给NoVNC服务器(此处为e.g:116.255.166.126:6080),而(NoVNC服务器上的)
  websockify 获取vnc_url中的token,由TokenDB查询token是否合法,如果合法,就将该token对应的['客户机host',
  '客户机vnc端口'](列表形式) 返回给 websockify 的反向代理功能组件 ,由其进行解析并反向代理,然后这个VNC_URL
  页面显示的就是 '客户机宿主IP':'客户机在该宿主上的vnc端口' 对应的视图了; that'a all

_ _ _

#### 视图功能实现原理 ####
1.页面传token值到 websockify

2.websockify 的 TokenDB 查询 vnc_tokens.db 中的 tokens 表获取 token对应的值(列表,
  包含ip,port)[`host_ip`,`kvm_vnc_port`]; tokens的表结构在 token_plugin.py中

3.websockify 为 host_ip:kvm_vnc_port 提供反向代理

4.而页面所持有的 token 是从 计算节点 的 CloudKvmApi 生成的.

5.CloudKvmApi 将前端传入的 客户机的名字换成 uuid,以及(由客户机名字查询出的)客户机的vnc端口,当前
  宿主的ip信息, 通过 http 请求调用 vnc 服务器的的 vnc_service(<VNC_SERVICE>:<9000>) 将 token
  信息 或插入 或(按客户机uuid)更新

#### 视图调用请求数据流 ####

Java接口调用CloudAPI:

开启VNC视图:

请求链接: http://10.160.0.6:9999

请求参数:
```
{
  "action":"openVNC",
  "module":"instance",
  "instance_name":"TESTcdrom"
}
```

成功响应:

```
{
   "token": "deba6913-d424-4709-972e-6c4b7d202543", 
   "code": 0, 
   "message": "SUCCESS"
}
```

失败响应:

```
{
   "code": 1, 
   "message": "Fails"
}
```

CloudAPI调用novnc_service

请求链接: http://10.160.0.113:9000

请求参数:
```
{
     "action":"gen_token",
     "guest_uuid":"f4b61ccb-f7b7-4c29-abea-10a329ca4ba1",
     "host":"10.160.0.6",
     "port":5914
}
```

成功响应:

```
{
    "token":"deba6913-d424-4709-972e-6c4b7d202543"
}
```
失败响应:
```
"exception:......"
```

如果上述还不够详细请联系作者: guojy8993@163.com
