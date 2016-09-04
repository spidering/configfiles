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

自動起動させる為に、systemctl enable nginxとsystemctl enable php-fpmをおこうなう。
/etc/nginx/conf.d/default.confに追記

    location ~\.php$ {
       fastcgi_pass 127.0.0.1:9001;
       fastcgi_index index.php;
       fastcgi_param SCRIPT_FILENAME /home/vagrant/www/html$fastcgi_script_name;
    }
    
###SPAWN-FCGIをインストール
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
/etc/nginx/conf.d/default.confのserverセクションに追記

    location ~ \.pl|cgi$ {
    fastcgi_pass 127.0.0.1:9001;
    fastcgi_index index.cgi;
    #$document_root$fast_cgi_script_nameとする記述もあるがうまくいかなかった
    fastcgi_param SCRIPT_FILENAME /home/vagrant/www/html$fast_cgi_script_name; 
    include       fastcgi_params;
    }
######Unix Socketで動かす場合

###SELINUXの設定を確認する。  
SELINUX=disabledにする

####その他
pearをインストール
PHP-FPMに直接関係があるわけではないが、phpenvでインストールするとpearが入らない(
インストールオプションが指定されていない為)
phpenv/plugins/php-build/share/php-build/default_configure_optionsにある
--without-pearを--with-pearに変更して再度インストール。
###参考URL
