#PHP-FPMをphpenv環境でインストールすると、php-fpmのパスが通ってないので、
設定する。
.zshenvに
PHP5_VER="5.6.25"
export PATH="/home/vagrant/.phpenv/versions/${PHP5_VER}/sbin:$PATH"
を追加。

#pearをインストール
PHP-FPMに直接関係があるわけではないが、phpenvでインストールするとpearが入らない(
インストールオプションが指定されていない為)
phpenv/plugins/php-build/share/php-build/default_configure_optionsにある
--without-pearを--with-pearに変更して再度インストール。

#php-fpmの起動スクリプトを作成
phpenvでPHPをインストールした際に作成されるディレクトリの中にある。
場所は、/tmp/php-build/source/5.6.25/sapi/fpm/php-fpm.service
これを/usr/lib/systemd/systemの中にコピーする。
ファイルを下記の様に書き換える。
[Unit]
Description=The PHP FastCGI Process Manager
After=syslog.target network.target

[Service]
Type=simple
PIDFile=/var/run/php-fpm.pid
ExecStart=/home/vagrant/.phpenv/versions/5.6.25/sbin/php-fpm --nodaemonize --fpm-config /home/vagrant/.phpenv/versions/5.6.25/etc/php-fpm.conf
ExecReload=/bin/kill -USR2 $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID


[Install]
WantedBy=multi-user.target

自動起動させる為に、systemctl enable nginxとsystemctl enable php-fpmをおこうなう。

#SPAWN-FCGIをインストール
sudo yum install --enablerepo=epel spawn-fcgi fcgi-devel
git clone https://github.com/gnosek/fcgiwrap.git fcgiwrap
cd fcgiwrap
autoreconf -i
./configure
make
sudo make install
vi /etc/sysconfig/spawn-fcgi
これを追記する
https://github.com/gnosek/fcgiwrap.git fcgi

#SELINUXの設定を確認する。
SELINUX=disabledにする