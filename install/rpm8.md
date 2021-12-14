# Установка Сертифицированного КриптоПро CSP 5.0 на сервер под управлением RHEL 8 based дистрибутивов

> ВСЕ ДЕЙСТВИЯ НЕОБХОДИМО ВЫПОЛНЯТЬ ОТ **ROOT**

> ПРЕДПОЛАГАЕМ, ЧТО ВСЕ ДЕЙСТВИЯ БУДУТ ВЫПОЛНЯТЬСЯ В ДОМАШНЕМ КАТАЛОГЕ ПОЛЬЗОВАТЕЛЯ **ROOT**

0. Заходим под суперпльзователем и идём в домашнюю директорию

```shell script
sudo -s
cd ~
```

1. Установливаем необходимые пакеты

```shell script
yum install -y php74-php-devel boost-devel gcc-c++ lsb libxml2-devel
```

2. Скачиваем архив с исходниками установленного PHP и распаковываем этот архив

```shell script
if ! rpm -q php >/dev/null 2>&1; then wget https://www.php.net/distributions/php-$( php -v | grep -Eo '[0-9\.]{1,20}' | head -n 1 ).tar.gz -O php_src.tar.gz; fi
tar xvf php_src.tar.gz
mv ~/php-$( php -v | grep -Eo '[0-9\.]{1,20}' | head -n 1 ) ~/php_src
``` 

3. Скачиваем архив с КриптоПро CSP 5.0 R2 для Linux RPM x64 с официального сайта [https://www.cryptopro.ru/products/csp/downloads](https://www.cryptopro.ru/products/csp/downloads), загружаем архив на сервер, распаковываем и устанавливаем
   минимальный набор пакетов КриптоПро CSP

> Скачивать нужно именно R2. R3 не собирается и проблему пока не решили!

```shell script
tar xvf linux-amd64.tgz
~/linux-amd64/install.sh
``` 

4. Устанавливаем пакет cprocsp-devel версии 5.0

```shell script
rpm -i ~/linux-amd64/lsb-cprocsp-devel*.rpm
``` 

5. Скачиваем архив с КриптоПро ЭЦП SDK, распаковываем этот архив и устанавливаем пакеты cprocsp-pki-cades, cprocsp-pki-phpcades

```shell script
wget https://cryptopro.ru/sites/default/files/products/cades/current_release_2_0/cades-linux-amd64.tar.gz
mkdir -p  ~/cades-linux-amd64
tar xvf cades-linux-amd64.tar.gz -C ~/cades-linux-amd64
rpm -i ~/cades-linux-amd64/cprocsp-pki-cades-64*.rpm ~/cades-linux-amd64/cprocsp-pki-phpcades-64*.rpm 
```

6. Скачиваем патч для PHP 7 и применяем его

```shell script
wget https://www.cryptopro.ru/sites/default/files/products/cades/php7_support.patch.zip
unzip -p ~/php7_support.patch.zip php7_support.patch > /opt/cprocsp/src/phpcades/php7_support.patch
cd /opt/cprocsp/src/phpcades/
patch -p0 < ./php7_support.patch
```

7. В файле Makefile.unix указываем путь к папке с исходниками php и вносим другие правки

```shell script
sed -i 's/PHPDIR=\/php/PHPDIR=\/root\/php_src/g' /opt/cprocsp/src/phpcades/Makefile.unix
sed -i 's/-fPIC -DPIC/-fPIC -DPIC -fpermissive/g' /opt/cprocsp/src/phpcades/Makefile.unix
```

8. Перейдём в директорию с исходниками PHP и выполним настройку конфигурации

```shell script
cd ~/php_src
./configure --prefix=/opt/php
```

> Если будет ругаться, то нужно доустановить недостающие пакеты. Скорее всего не будет хватать sqlite-devel

9. Перейдём в директорию с исходниками расширения и выполним команду сборки

```shell script
cd /opt/cprocsp/src/phpcades
eval `/opt/cprocsp/src/doxygen/CSP/../setenv.sh --64`; make -f Makefile.unix
```

10. Выведем путь к расширениям PHP:

```shell script
php -i | grep extension_dir
```

11. Создадим в директории с расширениями символическую ссылку на собранную библиотеку libphpcades.so

```shell script
ln /opt/cprocsp/src/phpcades/libphpcades.so /usr/lib64/php/modules/libphpcades.so
```

12. Добавляем libphpcades в php.ini

```shell script
echo "extension=libphpcades.so" > /etc/php.d/30-libphpcades.ini
```

13. Проверяем работу модуля

```shell script
php -f /opt/cprocsp/src/phpcades/test_extension.php
```

> Если выдал `Cannot find object or property. (0x80092004)TEST FAIL` - значит модуль подключился. Теперь нужно настроить сертификаты
