---
layout: post
title: Ubuntu 20.04 서버에 StrongSwan VPN 설치하기
date: 2022-12-10 20:29:02 +0900
category: VPN
---

예전에 우분투 18.04에 구축해서 잘 이용했던 적이 있는 StrongSwan을 서버에 설치 및 설정하는 방법을 기록하고자 한다.(우분투 버전은 업해서)<br />

우분투 20.04 버전이 준비가 되었다면 아래와 같이 설치를 진행한다.

```sh
sudo apt install strongswan strongswan-pki libcharon-extra-plugins libcharon-extauth-plugins libstrongswan-extra-plugins
```

<br />

디렉토리를 /etc/ipsec.d 하위 디렉토리 몇몇 구성에 맞추어 생성한 후 권한을 당사자에게만으로 제한한다.

```sh
mkdir -p ~/pki/{cacerts,certs,priavte}
chmod 700 ~/pki
```

<br />

루트 키는 4096-bit RSA 방식으로 생성하도록 하자.
Privacy Enhanced Mail(PEM) 파일은 Public Key Infrastructure(PKI) 파일 타입의 하나로 연결된 인증서 컨테이너 파일이다.(concatenated certificate container file)

```sh
ipsec pki --gen --type rsa --size 4096 --outform pem > ~/pki/private/ca-key.pem
```

<br />

생성한 키를 배정하여 루트 인증서를 만든다.(인증서 이름은 내키는 대로)

```sh
ipsec pki --self --ca --lifetime 3650 --in ~/pki/private/ca-key.pem --type rsa \
    --dn "CN=인증서이름" --outform pem > ~/pki/cacerts/ca-cert.pem
```

<br />

다음 만들 인증서는 클라이언트 측에서 방금 만든 인증서로 서버의 진위성 확인을 시도할 때 쓰이는 것이다.

```sh
ipsec pki --gen --type rsa --size 4096 --outform pem > ~/pki/private/server-key.pem
```

<br />

생성하였으면 지정해 준다.(앞에서도 나온 dn은 distinguished name, cn은 common name, san은 subject alternate name의 약자)

```sh
ipsec pki --pub --in ~/pki/private/server-key.pem --type rsa | ipsec pki --issue --lifetime 1825 \
    --cacert ~/pki/cacerts/ca-cert.pem --cakey ~/pki/private/ca-key.pem \
    --dn "CN=서버_도메인이나_IP" --san "서버_도메인이나_IP" \
    --flag serverAuth --flag ikeIntermediate --outform pem > ~/pki/certs/server-cert.pem
```

<br />

만든 것들을 /etc/ipsec.d로 옮겨주자

```sh
sudo cp -r ~/pki/* /etc/ipsec.d/
```

<br />

ipsec.conf 파일을 백업한 후에 nano나 vi 등 편한걸로 수정한다.

```sh
sudo mv /etc/ipsec.conf{,.original}
sudo nano /etc/ipsec.conf
```

<br />

다른 부분들은 [파라미터]를 참조해 설정해 주면 되고 leftid 부분 변경은 잊지말자

```
config setup
    charondebug="ike 1, knl 1, cfg 0"
    uniqueids=no

conn ikev2-vpn
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
    dpdaction=clear
    dpddelay=300s
    rekey=no
    left=%any
    leftid=@서버_도메인이나_IP
    leftcert=server-cert.pem
    leftsendcert=always
    leftsubnet=0.0.0.0/0
```

필자는 후술 하겠지만, 윈도우 설정을 잘 못해 애먹은 적이 있었다. 증상은 10분 이내로 연결이 끊기는 현상이었는데, 그때 당시 조사하면서 알아낸 참고 부분들이다.<br />
어떤 분이 [make-before-break](https://wiki.strongswan.org/issues/2610) 버그 리포팅을 하였는데, 참조하여 strongswan.conf 파일을 설정, 연결 해제 전에 구성하고 해제하는 방식을 적용해 볼 수도 있다.<br />
그 밖에 Dead Peer Detaction(dpd) 쪽을 많이 의심했어서 조금 기술하자면, dpdaction을 clear로 해놓으면 설정해 놓은 딜레이 즉, 위의 설정의 dpddelay가 300초이니 5분마다 요청 패킷 확인 후 확인 되지 아니하면 연결을 클리어 시킨다.
hold나 restart 등으로 설정 할 수 있는데 자세한 것은 마찬가지로 [파라미터]를 참조하면 된다.<br />
[keep_alive](https://wiki.strongswan.org/issues/2792) 또한 설정시 auto를 route로 설정하는 등 기타 설정에 주의하여야 한다.<br/>
uniqueids를 never로 설정해주면 자동 idle 드랍 방지가 된다, 커넥션 여러개 구성시 기존 연결이 드랍되는 경우에 적용하면 되는데, 그 당시 생각으로는 ca를 여러개 구성해서 다중으로 연결하는게 바람직하다고 생각했던 것 같다.
알아보려다가 만 부분도 있는데 road warrior 모드도 알아보면 좋을 듯싶다.<br />

설정을 바꾸면 항상 재시작하는 것 잊지 말 것!(재시작 방법은 바로 다음 아래에)

<br />

ipsec.secrets를 수정하여 서버키 명시와 아이디 패스워드를 기입한다.

```sh
sudo vi /etc/ipsec.secrets
```
```
:RSA "server-key.pem"
아이디 : EAP "비밀번호"
```

<br />

설정이 끝나면 재시작한다.
```sh
sudo ipsec rereadsecrets
sudo ipsec reload
sudo ipsec restart
```

<br />

방화벽에서 500, 4500의 udp를 허용해 주도록 하자<br />

여기선 기본적인 ufw로 진행하였으니 그에 따른 설정 방법이다.

```
sudo ufw allow 500,4500/udp
```

<br />

서버가 인터넷 연결에 사용하고 있는 네트워크 인터페이스를 특정해 준다.

```sh
ip route | grep default
```

<br />

출력된 결괏값의 예
```
default via **아이피 노출 방지** dev enp1s0 proto dhcp src **아이피 노출 방지** metric 100
```
dev와 proto 사이의 enp1s0가 인터페이스 이름이다.

<br />

이제 이름을 알았으니 라우팅과 IPSec 패킷 포워딩에 약간의 정책들을 넣자.
```sh
sudo nano /etc/ufw/before.rules
```

<br />

*filter 섹션 위와 아래에 아래와 같이 기입한다.(enp1s0 부분을 본인 것에 맞게 고쳐 넣을 것)<br />
*nat은 방화벽이 VPN 클라이언트와 인터넷의 트래픽을 변환 시켜주는 부분이고<br />
*mangle은 최대 패킷 세그먼트 크기를 조정해 VPN 클라이언트와의 문제를 미연에 방지한다.<br />
마지막 두 줄은 Encapsulating Security Payload(ESP)가 신뢰하지 않는 네트워크를 거칠 때 보안성 향상 역할을 하는데 방화벽으로 하여금 ESP 트래픽을 포워딩하게 하여 클라이언트가 연결이 가능하게 끔 만든다.
```
*nat
-A POSTROUTING -s 10.10.10.0/24 -o enp1s0 -m policy --pol ipsec --dir out -j ACCEPT
-A POSTROUTING -s 10.10.10.0/24 -o enp1s0 -j MASQUERADE
COMMIT

*mangle
-A FORWARD --match policy --pol ipsec --dir in -s 10.10.10.0/24 -o enp1s0 -p tcp -m tcp --tcp-flags SYN,RST SYN -m tcpmss --mss 1361:1536 -j TCPMSS --set-mss 1360
COMMIT

*filter
:ufw-before-input - [0:0]
:ufw-before-output - [0:0]
:ufw-before-forward - [0:0]
:ufw-not-local - [0:0]

-A ufw-before-forward --match policy --pol ipsec --dir in --proto esp -s 10.10.10.0/24 -j ACCEPT
-A ufw-before-forward --match policy --pol ipsec --dir out --proto esp -d 10.10.10.0/24 -j ACCEPT
```

<br />

이제 네트워크 커널 파라미터들을 바꿔 인터페이스들 간의 라우팅이 가능하게 만들어 주자.
해야 할 것은 IPv4 패킷 포워딩 설정과 Path MTU(Maximum Transmission Unit) discovery를 꺼서 패킷 세분화 문제를 예방하고 ICMP(Internet Control Message Protocol) redirects의 송수신을 제한하여 중간자 공격을 방지하는 것이다.
```sh
sudo nano /etc/ufw/sysctl.conf
```
```
# Uncomment this to allow this host to route packets between interfaces
net/ipv4/ip_forward=1

# Disable ICMP redirects. ICMP redirects are rarely used but can be used in
# MITM (man-in-the-middle) attacks. Disabling ICMP may disrupt legitimate
# traffic to those sites.
net/ipv4/conf/all/accept_redirects=0
net/ipv4/conf/all/send_redirects=0

# 문서 제일 하단에 기입
net/ipv4/ip_no_pmtu_disc=1
```

<br />

ufw를 재시작한다.
```sh
sudo ufw disable
sudo ufw enable
```

<br />

관리하기 쉽게 인증서를 현재 경로로 복사해 놓자
```sh
cat /etc/ipsec.d/cacerts/ca-cert/pem >> ca-cert.cer
```

<br />

SFTP 등으로 사용할 기기로 옮긴다. 본인은 PSCP를 사용하여 윈도우로 옮겨 사용했다.(pscp나 putty 사용법도 곧 올려보려... ~~시간이 언제 될려나~~)
우선 서버쪽에서 할일은 끝났고 이제 윈도우와 IOS기기에 적용하는 법이다.(곧 추가해서 올릴 계획) Mac환경에서도 단순하게 적용가능하나 현재 소유한 Mac이 고장나 하나를 새로 장만하면 그때 스크린샷까지 보강해서 올리려 한다.
안드로이드는 안타깝지만 써보질 않아서 잘 모르겠다. 얘기를 듣기론 안드로이드만 StrongSwan용 어플을 통해야한다고 한다.

<br />

알다시피 보통 여기저기 구글링을 많이하기에 당시에 적어 놓지 않은 사이트는 기억하기 힘들지만 거의 대부분은
[디지털 오션](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-ikev2-vpn-server-with-strongswan-on-ubuntu-20-04)을 참조해 만들었던 기억이 있고 따라서 크레딧으로 남긴다.

<br />

<!-- endnote -->
[파라미터]: https://wiki.strongswan.org/projects/strongswan/wiki/ConnSection