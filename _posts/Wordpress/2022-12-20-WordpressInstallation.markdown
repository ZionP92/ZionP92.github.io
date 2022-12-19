---
layout: post
title: Ubuntu 22.04 서버에 Wordpress 설치하기
date: 2022-12-20 02:20:38 +0900
category: Wordpress
---

최근에 워드프레스를 다시금 설치할 일이 생겨서 한김에 정리하고자 한다.
서버 대여만 해서 해보다가 이번엔 남는 데스크탑을 써서 우분투 서버 설치, 기본적인 ufw설정 및 공유기 포트 포워딩 등 몇 가지 추가로 한 건 안 비밀, 하지만 앞선 것들은 그닥 워드프레스 관련으로 적을게 없으니 패스<br />

우분투 22.04 버전이 준비가 되었다면 아래와 같이 새로운 패키지 확인하고 버전 업을 진행한다.

```sh
sudo apt update && sudo apt upgrade -y
```

<br />

서버로는 아파치를 올린다. Nginx도 언젠가 써보고 싶지만 아직 시도는 안 해봤다.

```sh
sudo apt install apache2 -y
```

<br />

데이터 베이스로는 마리아DB를 썼다.

```sh
sudo apt install mariadb-server mariadb-client -y
```

<br />

설치가 완료 되면 실행시켜 주자

```sh
sudo systemctl start mariadb
```

<br />

Active로 뜨는가 확인

```sh
sudo systemctl status mariadb
```

<br />

마리아DB 보안을 증강하도록 하자.

```sh
sudo mysql_secure_installation
```

위와 같이 치면, 루트 패스워드 설정이 뜨는데 나는 설정해 놓은게 있어서 그냥 패스했다. 
anonymous users 지우는 것과, remote root login 불허, 테스트 데이터 베이스 지우기, privilege tables 갱신 등등 다 Y를 입력하면 된다.

<br />

자 이제 PHP를 설치한다.

```sh
sudo apt install php -y
```

<br />

난 데스크탑을 싹 밀고 os부터 깔아서 wget도 깔아야 했으나 있으면 패스하자

```sh
sudo apt install wget -y
```

<br />

워드프레스는 언제든 상관없이 최신 버전을 latest.zip으로 제공하니 최신 버전을 찾아 헤매지 않아도 된다.
~~근데 설치 직후에 업데이트 알람이 있어서 했다...~~

```sh
wget https://wordpress.org/latest.zip
```

<br />

다운로드가 끝났으면 ls로 파일명 확인 후에 압축을 푼다. 어차피 latest.zip이겠지만<br/> unzip도 없다면 sudo apt install unzip으로 설치해 주자 

```sh
unzip latest.zip
```

<br />

압축 해제가 끝났으면 워드프레스 폴더로 진입한다.

```sh
cd wordpress
```

<br />

예전엔 디렉토리를 만들어야 했던 것 같은데... 있었다. 아무튼 혹시 모르니 기록

```sh
sudo mkdir /var/www/html
```

<br />

있었든 만들었든 다 복사해서 넣어주자

```sh
sudo cp -r * /var/www/html
```

<br />

해당 폴더로 진입 후 index.html을 지운다. 잘 복사 됐는지 확인하고 싶다면 wordpress 디렉토리랑 여기서 ls 한번씩 쳐보자. index.html을 지우고도 대상 디렉토리에서 같은 방식으로 알아서 확인

```sh
cd /var/www/html
```

```sh
sudo rm -rf index.html
```

<br />

cgi, cli 그리고 gd 패키지 설치를 진행한다.
각각 의미하는 바는 CGI: common gateway interface 웹서버와 php등 컴파일러와 인터프리터의 통신을 위한 규격, CLI 이건 설명할 필요 없고, GD는 그래픽 라이브러리이다.

```sh
sudo apt install php-mysql php-cgi php-cli php-gd -y
```

<br />

설치가 끝나면 아파치를 재시작해준다.

```sh
sudo systemctl restart apache2
```

<br />

www-data를 /var/www/의 소유자 및 그룹오너로 바꾼다. 콜론 앞이 유저이고 뒤가 그룹고 소유권 변경 명령어 chown에 붙은 옵션 -R은 recursive의 의미이다 즉, 그 아래 디렉토리와 파일 전부를 뜻한다고 할 수 있다.
/var/www/는 기본적으로 755일테니 모든 권한을 준다는 것이다.<br/>
755는 8진법으로 표기하는 권한으로써 유닉스 바탕의 운영체제들에서 쓰는 방식이다.
첫째 자리부터 소유자, 그룹에 속한 유저, 외부인이고 읽기는 4 쓰기는 2 실행은 1로써
7은 4 + 2 + 1 즉 읽고 쓰고 실행이가능하고 5는 4 + 1로 읽거나 실행만 가능한 권한이라 할 수 있다.

```sh
sudo chown -R www-data:www-data /var/www/
```

<br />

이제 IP를 확인하고 브라우저 주소창에 넣어 본다. 나 처럼 직접 구성해서 쓰는 서버라면 공유기로 포트 포워딩을 해줘야 외부에서 접속이 가능 할 것이다. 아이피가 만약 192.168.XXX.XXX 이런식으로 뜬다면 100% 내부 망 아이피이니 공유기를 쓴다면 DDNS를 이용하거나 고정아이피로 설정을 한 뒤 포트 포워딩을 해서 지정 포트로만 넘기거나 DMZ설정으로 해당 서버 기기에 대한 공유기의 방어 정책을 철회하고 연결을 지정해 줄 수 있다. 그리고 설정 시엔 왠만하면 그냥 MAC 주소를 쓰는 게 정신 건강에 좋다 내부망 IP가 자동 할당이 되고 바뀔 수가 있으니깐

```sh
hostname -I
```

<br />

이미 다 끝내고 나서 정리해보자고 생각한 거라 스크린 샷은 깜빡했다.
IP를 주소를 입력하고 들어가면 워드프레스 마크와 언어 선택창이 나온다.
그럼 정상 작동하는 것이니 데이터 베이스 지정을 하기 위해 설정을 시작한다.

```sh
sudo mysql -u root -p
```

<br />

루트 패스워드를 넣고 들어가면 MariaDB 커맨드 창을 볼 수 있을 것이다. 그럼 데이터 베이스를 만들도록 하자.

```sql
create database wordpress;
```

<br />

Querry OK, 1 row affected 같은 말이 뜨면 성공적으로 만들어진 것이다. 확인 하려면 show databases;라고 입력해서 데이터 베이스들을 출력해 볼 수 있다. 그럼 유저를 만들고 권한을 부여하자.

```sql
create user "MY_USER_NAME"@"%" identified by "MY_PASSWORD";
grant all privileges on wordpress.* to "MY_USER_NAME"@"%";
```

MY_USER_NAME 부분에 원하는 ID를 MY_PASSWORD는 원하는 비밀번호 그리고 권한 부여할 떄 데이터 베이스는 위에서 만든 wordpress이다. 원하는 다른 이름이 있다면 데이터 베이스 생성 시부터 다 바꿔주자.

<br />

마리아DB를 나온다.

```sql
exit
```

<br />

아까 그 브라우저 창으로 돌아가서 언어를 선택하고 계속 진행하다 보면 기입할 곳들이 있는 화면이 나오는데 차례대로 마리아DB에서 생성한 것들을 넣어 주면 된다.<br/><br/>
Database Name에 위에서 만든 wordpress를 기입하고
Username에 MY_USER_NAME
Password에 MY_PASSWORD
Database Host는 localhost 그대로 두자, 같은 서버 안에 만든 데이터 베이스니
Table Prefix는 같은 데이터 베이스로 워드프레스 여러 개 돌릴꺼면 변경하라고 하는데 보통은 하나만 만드니 그냥 wp_ 그대로 두자.<br/><br/>
제출버튼 누르고 진행하면 정보 입력창이 나온다.<br/><br/>
Site Title란에 대충 아무거나 우선 넣자, 꼭 본인 사이트 이름이 아니라도 나중에 바꿀 수 있으니 상관 없다.<br/>
Username 처음에 들어갈 어드민 계정 아이디 아무거나 넣고 진행하자 되도록 admin은 안쓰는 편이 공격 노출에 안전할 것이다.<br/>
Password는 알아서 넣으면 되고 Your Email은 잘 확인하고 틀리지 않게 기입한다.
Search engine viability는 체크하면 검색엔진들이 해당 사이트가 검색되게 하는 것을 저해시킨다는 것인데 본인 비밀 사이트 아니고서야 그냥 체크하지 말자.<br/>
마지막으로 인스톨 버튼을 누르면 설치가 끝난다.<br/><br/><br/>

이후 여러가지 부속적인 관리 페이지나 워드프레스 안에서 설정은 차차 다루겠지만 미리 첨언하자면 도메인은 최소한 구매해 놓는 것이 바람직한 것 같다.
공짜 도메인이나 이런 것들이 있는데, 주의 할 점은 대게의 경우에 사이트가 유명해지거나 하면 사용하지 못하게 하고 팔아먹는다든지 하는 경우들이 많이 있다고 들었다.
정신건강을 위해서 사는 게 좋고 또 .com이나 .co.kr같은 유명 도메인들은 면비는 못 본 것 같다.
