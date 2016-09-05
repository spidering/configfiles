##VagrantのCentOS7でNginx+PHP+Perlを動作させる
*動作環境
*Vagrant1.8.5
*centos7(Vagrant Cloudにある公式[centos7][linkref]
[linkref]:https://atlas.hashicorp.com/centos/boxes/7 "centos7")
*phpenv plenv pyenv rbenv

PHP-FPMをphpenv環境でインストールすると、php-fpmのパスが通ってないので、
設定する。phpenvでのインストールはここでは割愛
.zshenvに
PHP5_VER="5.6.25"
export PATH="/home/vagrant/.phpenv/versions/${PHP5_VER}/sbin:$PATH"
を追加。

###php-fpmの起動スクリプトを作成
phpenvでPHPをインストールした際に作成されるディレクトリの中にある。
場所は、/tmp/php-build/source/5.6.25/sapi/fpm/php-fpm.service
これを/usr/lib/systemd/systemの中にコピーする。
######/usr/lib/systemd/system/php-fpm.service
    [Unit]
    Description=The PHP FastCGI Process Manager
    After=syslog.target network.target  
    
    
    [Service]
    Type=simple  
    PIDFile=/var/run/php-fpm.pid  
    ExecStart=/home/vagrant/.phpenv/versions/5.6.25/sbin/php-fpm --nodaemonize --fpm-config   /home/vagrant/.phpenv/versions/5.6.25/etc/php-fpm.conf  
    ExecReload=/bin/kill -USR2 $MAINPID  
    ExecStop=/bin/kill -s QUIT $MAINPID  
    
    
    [Install]  
    WantedBy=multi-user.target  

[php-fpm.serviceダウンロード](https://raw.githubusercontent.com/spidering/configfiles/master/php-fpm.service)

自動起動させる為に、systemctl enable nginxとsystemctl enable php-fpmをおこうなう。
/etc/nginx/conf.d/default.confに追記

    location ~\.php$ {
       fastcgi_pass 127.0.0.1:9001;
       fastcgi_index index.php;
       fastcgi_param SCRIPT_FILENAME /home/vagrant/www/html$fastcgi_script_name;
    }
    
###SPAWN-FCGIとfcgiwrapをインストール
    sudo yum install --enablerepo=epel spawn-fcgi fcgi-devel  
    git clone https://github.com/gnosek/fcgiwrap.git fcgiwrap  
    cd fcgiwrap  
    autoreconf -i  
    ./configure  
    make  
    sudo make install  


######TCP Socketで動かす場合
/etc/sysconfig/spawn-fcgiの最終行に

    OPTIONS="-u nginx -g nginx -a 127.0.0.1 -p 9001 -P /var/run/spawn-fcgi.pid -- /usr/local/sbin/fcgiwrap" を追加。
    # You must set some working options before the "spawn-fcgi" service will work.
    # If SOCKET points to a file, then this file is cleaned up by the init script.
    #
    # See spawn-fcgi(1) for all possible options.
    #
    # Example :
    #SOCKET=/var/run/php-fcgi.sock
    #OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"
    OPTIONS="-u nginx -g nginx -a 127.0.0.1 -p 9001 -P /var/run/spawn-fcgi.pid -- /usr/local/sbin/fcgiwrap" ←これを追記
[spawn-fcgi-tcpsocketダウンロード](https://raw.githubusercontent.com/spidering/configfiles/master/init.d/spawn-fcgi-tcpsocket)

/etc/nginx/conf.d/default.confのserverセクションに追記

    location ~ \.pl|cgi$ {
    fastcgi_pass 127.0.0.1:9001;
    fastcgi_index index.cgi;
    #$document_root$fast_cgi_script_nameとする記述もあるがうまくいかなかった
    fastcgi_param SCRIPT_FILENAME /home/vagrant/www/html$fast_cgi_script_name; 
    include       fastcgi_params;
    }
######Unix Socketで動かす場合
/etc/sysconfig/fcgiwrapの設定

    FCGI_USER=vagrant
    FCGI_GROUP=vagrant
    FCGI_PROGRAM=/usr/local/sbin/fcgiwrap
    FCGI_SOCKET=/var/run/fcgiwrap/fcgiwrap.sock
    FCGI_EXTRA_OPTIONS="-M 0700"
    OPTIONS="-u $FCGI_USER -g $FCGI_GROUP -s $FCGI_SOCKET -S $FCGI_EXTRA_OPTIONS -F 1 -P -$FCGI_PID -- $FCGI_PROGRAM"
[fcgiwrapダウンロード](https://raw.githubusercontent.com/spidering/configfiles/master/fcgiwrap)

/etc/init.d/spawn-fcgiの修正
下記の部分をコメントアウトまたは削除。ここではコメントアウトを既にしている状態
    
    #exec="/usr/bin/spawn-fcgi"
    #cgi="/usr/local/sbin/fcgiwrap"
    #config="/etc/sysconfig/spawn-fcgi"
    続いて下記をを追加
    
    exec="/usr/bin/spawn-fcgi"
    cgi="/usr/local/sbin/fcgiwrap"
    prog="`basename $cgi`
    config="/etc/sysconfig/$prog"
    pid="/var/run/fcviwrap/spawn-fcgi.pid"
    SOCKET="/var/run/fcgiwrap/fcgiwrap.sock"
    
    続いてstop()項目にある
    #killproc $prog コメントアウトコメントアウトしてから
    killproc -p $pid $prog を追加
[spawn-fcgi-unixsocketダウンロード](https://raw.githubusercontent.com/spidering/configfiles/master/init.d/spawn-fcgi-unixsocket)

nginxの設定を変更する

    location ~\.pl|cgi$ {
    fastcgi_index index.pl;
    fastcgi_pass unix:/var/run/fcgiwrap/fcgiwrap.sock
    fastcgi_param SCRIPT_FILENAME /home/vagrant/www/html$fastcgi_script_name;
    include       fastcgi_params;
    }
###SELINUXの設定を確認する。  
SELINUX=disabledにする

####その他
pearをインストール
PHP-FPMに直接関係があるわけではないが、phpenvでインストールするとpearが入らない(
インストールオプションが指定されていない為)
phpenv/plugins/php-build/share/php-build/default_configure_optionsにある
--without-pearを--with-pearに変更して再度インストール。
###参考URL
