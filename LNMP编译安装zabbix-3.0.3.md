# LNMP编译安装zabbix-3.0.3

软件本：
	zabbix	：zabbix-3.0.3
	MySQL	：mariadb-10.1.14.tar.gz
	PHP		：php-7.0.7
	Nginx	：nginx-1.11.1
	
## 一、安装mariadb

编译安装MariaDB-10.1.14

### 1、安装必要依赖包
	yum -y install gcc gcc-c++ make cmake ncurses ncurses libxml2 libxml2-devel openssl-devel bison bison-devel  ncurses-devel

### 2、下载解压
	tar -xf mariadb-10.1.14-linux-x86_64.tar.gz

### 3、创建数据存放目录
	mkdir -p /data

### 4、创建用户
	useradd mysql -s /sbin/nologin -M

### 5、变更属主
	chown -R mysql:mysql /data

### 6、编译安装
	cmake . -DCMAKE_INSTALL_PREFIX=/application/mariadb-10.1.14 \
	        -DMYSQL_DATADIR=/data \
	        -DMYSQL_UNIX_ADDR=/data/tmp/mysql.sock \
	        -DDEFAULT_CHARSET=utf8 \
	        -DDEFAULT_COLLATION=utf8_general_ci \
	        -DEXTRA_CHARSETS=gbk,gb2312,utf8,ascii \
	        -DENABLED_LOCAL_INFILE=ON \
	        -DWITH_INNOBASE_STORAGE_ENGINE=1 \
	        -DWITH_FEDERATED_STORAGE_ENGINE=1 \
	        -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
	        -DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 \
	        -DWITHOUT_PARTITION_STORAGE_ENGINE=1 \
	        -DWITH_FAST_MUTEXES=1 \
	        -DWITH_ZLIB=bundled \
	        -DENABLED_LOCAL_INFILE=1 \
	        -DWITH_READLINE=1 \
	        -DWITH_EMBEDDED_SERVER=1 \
	        -DWITH_DEBUG=0

### 7、安装
	make && make install

### 8、建立软连接
	ln -s /application/mariadb-10.1.14 /application/mysql

### 9、初始化数据库
	/application/mysql/scripts/mysql_install_db --basedir=/application/mysql --datadir=/data/ --user=mysql

### 10、生成配置文件
	mv /etc/my.cnf /etc/my.cnf.bak
	cp support-files/my-large.cnf /etc/my.cnf

### 11、添加环境变量
	echo 'PATH="/application/mariadb-10.1.14/bin/:$PATH"' >>/etc/profile

### 12、添加启动文件
	cp /application/mariadb-10.1.14/support-files/mysql.server /etc/init.d/mysqld

### 13、添加自启动
	cat /usr/lib/systemd/system/mysqld.service 
		[Unit]  
		Description=mysqld
		After=network.target  
		
		[Service]  
		Type=forking  
		ExecStart=/etc/init.d/mysqld start  
		ExecReload=/etc/init.d/mysqld restart  
		ExecStop=/etc/init.d/mysqld  stop  
		PrivateTmp=true  
		
		[Install]  
		WantedBy=multi-user.target
	
	systemtl enable mysqld.service

### 14、启动MySQL
	systemctl start mysqld.service

## 二、安装Nginx（使用一键安装脚本安装）

	#!/bin/bash
	#
	# nginx         : Auto install Nginx.
	# chkconfig     : 
	# description   : Auto install Nginx.
	
	### BEGIN INIT INFO
	# Provides:
	# Should-Start:
	# Short-Description: Auto install nginx.
	# Description: Auto install nginx.
	### END INIT INFO
	
	# There are the dependent packages that you have to install first.
	yum install pcre pcre-devel openssl-devel wget -y
	
	USER="www"
	NGINX="nginx-1.11.1"
	DIR="/application"
	DOWNLOADURL="http://nginx.org/download/nginx-1.11.1.tar.gz"
	
	# Install Nginx
	if [ ! -d "$DIR/tool" ]
	   then
	        mkdir -p $DIR/tools
	   else
	        echo "Directory already exists."
	fi
	cd $DIR/tools
	id $USER &>/dev/null
	RETVAL=$?
	if [ $RETVAL -ne 0 ]
	   then 
	        useradd $USER -s /sbin/nologin -M
	        echo "Creat $USER successfully."
	   else
	        echo "User $SUER already exists."
	fi
	if [ ! -f "$NGINX.tar.gz" ]
	   then
	        wget $DOWNLOADURL
	        tar -zxf $NGINX.tar.gz
	   else
	        tar -zxf $NGINX.tar.gz
	fi
	cd $NGINX
	        ./configure --prefix=$DIR/$NGINX \
	                    --user=$USER \
	                    --group=$USER \
	                    --with-http_ssl_module \
	                    --with-http_stub_status_module
	make && make install
	cd $DIR
	ln -s $DIR/$NGINX $DIR/nginx
	
	echo $'#!/bin/sh
	#
	# nginx - this script starts and stops the nginx daemon
	#
	# chkconfig:   - 85 15
	# description:  NGINX is an HTTP(S) server, HTTP(S) reverse \
	#               proxy and IMAP/POP3 proxy server
	# processname: nginx
	# config:      /application/nginx/conf/nginx.conf
	# config:      /etc/sysconfig/nginx
	# pidfile:     /application/nginx/logs/nginx.pid
	# Source function library.
	. /etc/rc.d/init.d/functions
	# Source networking configuration.
	. /etc/sysconfig/network
	# Check that networking is up.
	[ "$NETWORKING" = "no" ] && exit 0
	nginx="/application/nginx/sbin/nginx"
	prog=$(basename $nginx)
	NGINX_CONF_FILE="/application/nginx/conf/nginx.conf"
	[ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx
	lockfile=/var/lock/subsys/nginx
	make_dirs() {
	   # make required directories
	   user=`$nginx -V 2>&1 | grep "configure arguments:" | sed \'s/[^*]*--user=\\([^ ]*\\).*/\\1/g\' -`
	   if [ -z "`grep $user /etc/passwd`" ]; then
	       useradd -M -s /bin/nologin $user
	   fi
	   options=`$nginx -V 2>&1 | grep \'configure arguments:\'`
	   for opt in $options; do
	       if [ `echo $opt | grep \'.*-temp-path\'` ]; then
	           value=`echo $opt | cut -d "=" -f 2`
	           if [ ! -d "$value" ]; then
	               # echo "creating" $value
	               mkdir -p $value && chown -R $user $value
	           fi
	       fi
	   done
	}
	start() {
	    [ -x $nginx ] || exit 5
	    [ -f $NGINX_CONF_FILE ] || exit 6
	    make_dirs
	    echo -n $"Starting $prog: "
	    daemon $nginx -c $NGINX_CONF_FILE
	    retval=$?
	    echo
	    [ $retval -eq 0 ] && touch $lockfile
	    return $retval
	}
	stop() {
	    echo -n $"Stopping $prog: "
	    killproc $prog -QUIT
	    retval=$?
	    echo
	    [ $retval -eq 0 ] && rm -f $lockfile
	    return $retval
	}
	restart() {
	    configtest || return $?
	    stop
	    sleep 1
	    start
	}
	reload() {
	    configtest || return $?
	    echo -n $"Reloading $prog: "
	    killproc $nginx -HUP
	    RETVAL=$?
	    echo
	}
	force_reload() {
	    restart
	}
	configtest() {
	  $nginx -t -c $NGINX_CONF_FILE
	}
	rh_status() {
	    status $prog
	}
	rh_status_q() {
	    rh_status >/dev/null 2>&1
	}
	case "$1" in
	    start)
	        rh_status_q && exit 0
	        $1
	        ;;
	    stop)
	        rh_status_q || exit 0
	        $1
	        ;;
	    restart|configtest)
	        $1
	        ;;
	    reload)
	        rh_status_q || exit 7
	        $1
	        ;;
	    force-reload)
	        force_reload
	        ;;
	    status)
	        rh_status
	        ;;
	    condrestart|try-restart)
	        rh_status_q || exit 0
	            ;;
	    *)
	        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
	        exit 2
	esac'>/etc/init.d/nginxd
	
	chmod +x /etc/init.d/nginxd
	
	echo $'[Unit]  
	Description=nginx  
	After=network.target  
	
	[Service]  
	Type=forking  
	ExecStart=/etc/init.d/nginxd start  
	ExecReload=/etc/init.d/nginxd restart  
	ExecStop=/etc/init.d/nginxd  stop  
	PrivateTmp=true  
	
	[Install]  
	WantedBy=multi-user.target'>/lib/systemd/system/nginx.service
	
	chmod +x /lib/systemd/system/nginx.service
	
	systemctl enable nginx.service
	systemctl start nginx.service

## 三、编译安装ＰＨＰ-7.0.7

编译安装PHP-7.0.7
### 1、安装epel源
	rpm -ivh http://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
安装epel源是为了解决依赖包libmcrypt安装

### 2、安装依赖包
	yum install -y zlib libxml libjpeg freetype libpng  gd curl libiconv zlib-devel libxml2-devel libjpeg-devel freetype-devel libpng-devel gd-devel curl-devel openssl openssl-devel libmcrypt-devel mhash mhash-devel mcrypt libxslt*

### 3、安装libiconv
	wget http://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.14.tar.gz
	tar -zxvf libiconv-1.14.tar.gz
	cd  libiconv-1.14
	./configure --prefix=/usr/local/libiconv
	make && make install

报错及解决方法：

	make  all-am
	make[2]: Entering directory `/application/tools/libiconv-1.14/srclib'
	make[3]: Entering directory `/application/tools/libiconv-1.14'
	make[3]: Nothing to be done for `am--refresh'.
	make[3]: Leaving directory `/application/tools/libiconv-1.14'
	gcc -DHAVE_CONFIG_H -DEXEEXT=\"\" -I. -I.. -I../lib  -I../intl -DDEPENDS_ON_LIBICONV=1 -DDEPENDS_ON_LIBINTL=1   -g -O2 -c allocator.c
	gcc -DHAVE_CONFIG_H -DEXEEXT=\"\" -I. -I.. -I../lib  -I../intl -DDEPENDS_ON_LIBICONV=1 -DDEPENDS_ON_LIBINTL=1   -g -O2 -c areadlink.c
	gcc -DHAVE_CONFIG_H -DEXEEXT=\"\" -I. -I.. -I../lib  -I../intl -DDEPENDS_ON_LIBICONV=1 -DDEPENDS_ON_LIBINTL=1   -g -O2 -c careadlinkat.c
	gcc -DHAVE_CONFIG_H -DEXEEXT=\"\" -I. -I.. -I../lib  -I../intl -DDEPENDS_ON_LIBICONV=1 -DDEPENDS_ON_LIBINTL=1   -g -O2 -c malloca.c
	gcc -DHAVE_CONFIG_H -DEXEEXT=\"\" -I. -I.. -I../lib  -I../intl -DDEPENDS_ON_LIBICONV=1 -DDEPENDS_ON_LIBINTL=1   -g -O2 -c progname.c
	In file included from progname.c:26:0:
	./stdio.h:1010:1: error: ‘gets’ undeclared here (not in a function)
	 _GL_WARN_ON_USE (gets, "gets is a security hole - use fgets instead");
	 ^
	make[2]: *** [progname.o] Error 1
	make[2]: Leaving directory `/application/tools/libiconv-1.14/srclib'
	make[1]: *** [all] Error 2
	make[1]: Leaving directory `/application/tools/libiconv-1.14/srclib'
	make: *** [all] Error 2

解决方法：
	进入srclib目录 执行 sed -i -e '/gets is a security/d' ./stdio.in.h

	make && make install

### 4、生成配置文件
	./configure \
	--prefix=/application/php-7.0.7 \
	--with-mysqli=mysqlnd \
	--with-pdo-mysql=mysqlnd \
	--with-iconv-dir=/usr/local/libiconv \
	--with-freetype-dir \
	--with-jpeg-dir \
	--with-png-dir \
	--with-zlib \
	--with-libxml-dir=/usr \
	--enable-xml \
	--disable-rpath \
	--enable-bcmath \
	--enable-shmop \
	--enable-sysvsem \
	--enable-inline-optimization \
	--with-curl \
	--enable-mbregex \
	--enable-fpm \
	--enable-mbstring \
	--with-mcrypt \
	--with-gd \
	--enable-gd-native-ttf \
	--with-openssl \
	--with-mhash \
	--enable-pcntl \
	--enable-sockets \
	--with-xmlrpc \
	--enable-zip \
	--enable-soap \
	--enable-short-tags \
	--enable-static \
	--with-xsl \
	--with-fpm-user=www \
	--with-fpm-group=www \
	--enable-ftp \
	--with-gettext

生成配置文件时提示：
	
	configure: WARNING: unrecognized options: --with-mysql, --enable-safe-mode, --with-curlwrappers, --enable-zend-multibyte		##不用的编译项，不用管

	make && make install

### 5、生成配置文件
	cp /application/tools/php-7.0.7/php.ini-production /application/php/lib/php.ini


### 6、创建软连接
	ln -s /application/php-7.0.7 /application/php

### 7、生成php-fpm配置文件
	cp /application/php/etc/php-fpm.conf.default /application/php/etc/php-fpm.conf
	cp /application/php/etc/php-fpm.d/www.conf.default /application/php/etc/php-fpm.d/www.conf

### 8、生成启动脚本

	#!/bin/bash
	# php-fpm startup script for the php-fpm 
	# php-fpm version:5.5.0-alpha6
	# chkconfig: - 85 15
	# description: php-fpm is very good
	# processname: php-fpm
	# pidfile: /application/php/var/run/php-fpm.pid
	# config: /application/php/php/etc/php-fpm.conf
	  
	php_command=/application/php/php/sbin/php-fpm
	php_config=/application/phpl/php/etc/php-fpm.conf
	php_pid=/application/php/var/run/php-fpm.pid
	RETVAL=0
	prog="php-fpm"
	  
	#start function
	php_fpm_start() {
	    $php_command
	}
	  
	start(){
	    if [ -e $php_pid  ]
	    then
	    echo "php-fpm already start..."
	    exit 1
	    fi
	    php_fpm_start
	}
	  
	stop(){
	    if [ -e $php_pid ]
	    then
	    parent_pid=`cat $php_pid`
	    all_pid=`ps -ef | grep php-fpm | awk '{if('$parent_pid' == $3){print $2}}'`
	    for pid in $all_pid
	    do
	            kill $pid
	        done
	        kill $parent_pid
	    fi
	    exit 1
	}
	  
	restart(){
	    stop
	    start
	}
	  
	# See how we were called.
	case "$1" in
	start)
	        start
	        ;;
	stop)
	        stop
	        ;;
	restart)
	        stop
	        start
	        ;;
	status)
	        status $prog
	        RETVAL=$?
	        ;;
	*)
	        echo $"Usage: $prog {start|stop|restart|status}"
	        exit 1
	esac
	exit $RETVAL

### 9、测试
测试LNMP安装情况：	#将以下代码保存到Nginx网页安装目录
#### 1、配置nginx.conf
添加：
	location ~ .*\.(php|php7)?$ {
	             root   /application/nginx/html/zabbix;
	             fastcgi_pass   127.0.0.1:9000;
	             fastcgi_index  index.php;
	             include        fastcgi.conf;
	        }

#### 2、测试PHP安装情况
	<?php
	   phpinfo();
	?>

### 3、测试PHP连接MySQL
	 cat /application/nginx/html/zabbix/index.php 
	<?php
	$dbh = new PDO('mysql:host=localhost;dbname=mysql;port=3306','root','abc123');
	$query = "select user,host from mysql.user";  
	$result = $dbh->query($query);  
	$rows = $result->fetchAll();  
	foreach($rows as $row)  
	{  
	$ym = $row[0];  
	$user = $row[1];  
	$ps = $row[2];  
	printf("ym-".$ym." user-".$user." ps-".$ps);  
	print("</br>");  
	}  
	  
	$dbh = null;
	?>

	注意：配置为localhost时不能连接MySQL需要修改php.ini文件
	pdo_mysql.default_socket = /data/tmp/mysql.sock

## 四、编译安装zabbix-3.0.3

编译安装Zabbix-3.0.3
### 1、解压
	tar -zxvf zabbix-3.0.3.tar.gz

### 2、建立用户
	useradd zabbix

### 3、安装依赖包
	yum install -y net-snmp net-snmp-devel libcurl-devel

### 4、编译
	./configure --prefix=/application/zabbix-3.0.3 --enable-server --enable-agent --with-mysql --enable-ipv6 --with-net-snmp --with-libcurl --with-libxml2

### 5、安装
	make install

### 6、建立软链接
	ln -s /application/zabbix-3.0.3 /application/zabbix

### 7、修改php.ini

	vi /application/php/lib/php.ini #编辑修改
	
	post_max_size =16M
	
	max_execution_time =300
	
	max_input_time =300

### 8、配置自启动

	cp /application/tools/zabbix-3.0.3/misc/init.d/fedora/core/zabbix_server /etc/init.d/zabbix-server		##修改路径等

	vim /usr/lib/systemd/system/zabbix-server.service
	[Unit]  
	Description=zabbix_server
	After=network.target  
	
	[Service]  
	Type=forking  
	ExecStart=/etc/init.d/zabbix_server start  
	ExecReload=/etc/init.d/zabbix_server restart  
	ExecStop=/etc/init.d/zabbix_server  stop  
	PrivateTmp=true  
	
	[Install]  
	WantedBy=multi-user.target

### 9、启动

	systemctl enable zabbix-server.service
	systemctl start zabbix-server


 