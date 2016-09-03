PHP-FPMをphpenv環境でインストールすると、php-fpmのパスが通ってないので、
設定する。

.zshenvに
PHP5_VER="5.6.25"
export PATH="/home/vagrant/.phpenv/versions/${PHP5_VER}/sbin:$PATH"
を追加。

PHP-FPMに直接関係があるわけではないが、phpenvでインストールするとpearが入らない(
インストールオプションが指定されていない為)
phpenv/plugins/php-build/share/php-build/default_configure_optionsにある
--without-pear隣っている部分を--with-pearに変更して再度インストール。

起動スクリプトはphpenvでPHPをインストールした際に作成されるディレクトリの中にある。
場所は、/tmp/php-build/source/5.6.25/sapi/fpm/php-fpm.service
これを/usr/lib/systemd/systemの中にコピーする。
これを元に多少の変更。

自動起動させる為に、systemctl enable nginxとsystemctl enable php-fpmをおこうなう。