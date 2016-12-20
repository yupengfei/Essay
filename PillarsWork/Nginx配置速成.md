#Nginx配置速成

## 安装nginx

## 基本命令

	nginx -s signal

其中signal可以为下列之一

	1. stop 快速停止
	1. quit 优雅停止
	1. reload 重载配置文件，使配置文件生效，它会检查配置文件是否正确，如果正确使用新配置，否则使用旧配置
	1. reopen 重新打开log文件

## 配置文件

nginx的配置文件由指令组成，有些指令可以使用括号包含其它指令，被称为上下文，如events、http、server、location。不被别的上下文包含的指令叫全局上下文。

## 静态文件

假设我们的html文件在/data/www文件夹内，图片在/data/images文件夹内。

	location / {
		root /data/www;
	}

这时，所有/开头的请求会被添加上root指令指定的地址，即/data/www，组成最终的地址/data/www/。

如果有多个location都与url匹配，nginx会匹配最长的location，因此，如果其它所有的location都匹配不上，就会匹配这个/。

对于图片，增加另一个location。

	location /images/ {
		root /data;
	}

这样，所有以/images开头的请求都被映射到/data/images/。

## 代理服务器地址

我们配置一个将图片的请求映射到本地存储，其它的请求转到其它的服务器。

	server {
		listen 8080
		root /data/up1;

		location / {

		}
	}

这个是一个简单的服务器，监听8080端口，将所有的请求映射到/data/up1目录。注意此时的root在server的上下文中，所以如果location没有指定root指令的话，默认会用server下面的root。

我们做一个简单的修改，增加一个proxy_pass指令，并指定协议类型，服务器地址和端口。

	server {
		location / {
			proxy_pass http://localhost:8080/;
		}

		location /images/ {
			root /data;
		}
	}

这样我们就把所有url里面带/images/前缀的请求映射到/data/iamges/，其余的请求转发给http://localhost:8080/。我们也可以使用正则表达式把所有特定后缀的请求映射到对应的路径。

	location ~ \.(gif|jpg|png)$ {
		root /data/images;
	}

匹配规则如下，nginx先匹配所有的location路径前缀，选取最长的那个，然后再匹配正则表达式，如果有正则表达式匹配，则使用正则表达式，否则使用最长的那个location匹配。

我们得到

	server {
		location / {
			proxy_pass http://localhost:8080/;
		}

		location ~ \.(gif|jpg|png)$ {
			root /data/images;
		}
	}

这个配置将以.gif, .jpg, .png结尾的请求映射到/data/images文件夹，其余的请求转发到代理服务器。

