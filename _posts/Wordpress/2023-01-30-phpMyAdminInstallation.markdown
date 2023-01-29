---
layout: post
title: phpMyAdmin 설치하기
date: 2023-01-30 06:47:00 +0900
category: Wordpress
---

아무래도 워드프레스 사이트를 관리하기에 필요하여 설치를 진행하였다. 저번에 설치한 워드프레스 서버에 설치를 진행하는 만큼 이미 설치된 것은 생략된 부분도 있음을 먼저 알린다. 환경은 이어서 진행하니 ubuntu 22.04에 php 8.1.2 그대로이다.<br />

전에 설치하지 않은 필요 패키지들을 설치해준다. 나머지는 이름 그대로이고 mbstring의 경우엔 ASCII가 아닌 string들을 처리해주는 기능을 한다. 

```sh
sudo apt install php-zip php-json php-mbstring
```

<br />

설치가 끝나면 분홍색 화면이 뜨면서 재시작할 기능들을 선택하라고 하는데 그냥 그대로 진행하였다.

<br />

그냥 phpMyAdmin을 apt install phpmyadmin으로 받으면 구버전이 받아진다. [다운로드 페이지](https://www.phpmyadmin.net/downloads/)에 가서 받아도 되지만 나는 아래와 같이 진행하였다.

```sh
wget https://files.phpmyadmin.net/phpMyAdmin/5.2.0/phpMyAdmin-5.2.0-all-languages.zip 
unzip phpMyAdmin-5.2.0-all-languages.zip 
sudo mv phpMyAdmin-5.2.0-all-languages /usr/share/phpmyadmin 
```

<br />

옮기기까지 끝났으면 tmp 디렉토리를 생서하고 권한을 지정한다.

```sh
sudo mkdir /usr/share/phpmyadmin/tmp 
sudo chown -R www-data:www-data /usr/share/phpmyadmin 
sudo chmod 777 /usr/share/phpmyadmin/tmp 
```

<br />

웹서버가 phpMyAdmin을 다룰 수 있게 설정하기위해 conf파일에 진입한다.

```sh
sudo nano /etc/apache2/conf-available/phpmyadmin.conf 
```

<br />

다음과 같이 기입하고 사용하는 툴에 따라 알아서 저장하고 빠져나오면 된다.

```sh
Alias /phpmyadmin /usr/share/phpmyadmin
Alias /phpMyAdmin /usr/share/phpmyadmin
 
<Directory /usr/share/phpmyadmin/>
   AddDefaultCharset UTF-8
   <IfModule mod_authz_core.c>
      <RequireAny>
      Require all granted
     </RequireAny>
   </IfModule>
</Directory>
 
<Directory /usr/share/phpmyadmin/setup/>
   <IfModule mod_authz_core.c>
     <RequireAny>
       Require all granted
     </RequireAny>
   </IfModule>
</Directory>
```

<br />

스크립트를 사용하게 한 후 아파치를 재시작한다.

```sh
sudo a2enconf phpmyadmin 
sudo systemctl restart apache2 
```

<br />

이제 "서버_아이피_혹은_도메인/phpmyadmin" 주소로 진입하면 워드프레스 처럼 아이디 비밀번호 기입창이 나오는데 저번에 워드프레스에서 설정한 어드민 계정으로 진입이 가능하다.

<br />

이미 워드프레스 설치하면서 여타 잡다한 부분은 다 구성이 되어있기에 더욱이 간단하게 끝났다. phpMyAdmin은 잘못 해놓으면 사이트가 굉장히 취약해지니 잘 유의해서 사용해야 한다.