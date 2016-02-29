---
keywords: 2014, 总结
description:
---
{% if page.description %}
>{{ page.description }}
{% endif %}

## 安装nginx lua模块
&#12288;&#12288;如果已安装nginx，则需要使用源代码进行重新编译，加入新的模块得到二进制nginx替换原有的nginx文件即可（注意备份），中间遇到什么configure报错多半是由于某个库没有安装，下载对应的压缩文件（`wget`），解压（`tar`），然后配置（`./configure`）编译(`make  -j2`)安装(`make install`)即可，涉及到的库略多，安装了挺多才搞定，比如libxml2、libxslt、libcrypt等等。

## 配置脚本
&#12288;&#12288;nginx配置文件中匹配地址加入content\_by\_lua 内容则为lua命令，执行某段脚本（`os.execute('xxx.sh')`），但是遇到了权限的问题，除了不能创建文件夹，还产生了好几个莫名其妙的问题，比如git pull遇到`fatal: could not read Username for 'xxxxx': No such device or address`(同样是因为权限)，执行ruby脚本的时候，ruby库有函数报错（缺少环境变量）等等

## 遇到的一些问题

### nginx lua执行系统脚本的权限问题

&#12288;&#12288;os.execute需要有执行权限，使用io.popen代替，还可以读取脚本的输出结果，local t = io.open 中打印出当前用户发现是www-data ，即nginx启动的用户为www-data ，这里为了方便修改nginx.conf中的user为root ，但存在安全隐患，应当为www-data用户分配项目目录权限及脚本执行权限。

``` shell
server {
	listen 80;
	server_name www.xxx.cn xxx.cn;
	location = /deploy {
        content_by_lua ' 
            ngx.header.content_type = "text/plain";
            local t = io.popen("/xxx/deploy.sh");
            ngx.say(t:read("*all"));
	    ';
	}
}
```

### 执行脚本时读取不到环境变量

&#12288;&#12288;ruby库运行报错，经过定位库中代码，发现是未读取到环境变量导致报错，可能是环境变量设置时没有定为全局，在shell脚本中自行export环境变量临时解决了该问题

``` shell
export LANGUAGE=en_US:
export LANG=en_US.UTF-8
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

## 小结
&#12288;&#12288;过程中遇到问题多与权限相关，使用hook免去了部署过程，做程序猿还是要懂得如何偷懒，当然如果涉及到正规的团队项目，实行自动部署还是要在安全等各方面多加考虑。
