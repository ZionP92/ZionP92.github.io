---
layout: post
title: StrongSwan을 Windows와 iOS에서 사용하는 방법
date: 2023-03-27 16:08:06 +0900
category: VPN
---

저번에 만든 VPN의 사용 법이다. MacOS나 Android는 나중에 기회가 되면 올릴 생각이다.

시작하기에 앞서 StrongSwan VPN 서버 설치가 되어 있어야 하며 앞서 끝에 복사까지 해둔 ca-cert.cer 파일을 사용할 기기로 받아야 한다.

<br />

### Windows

pscp 등으로 ca-cert.cer 파일을 받도록 하자.

받은 인증서를 실행을 하여 설치를 진행해 준다.

아래는 인증서 설치를 누른 후 중요한 부분의 스크린 샷이다 그 다음은 끝까지 진행하여 주면 된다.

<img src="../../../../img/VPN/strongswan_cert_win_001.png" width=500>
<img src="../../../../img/VPN/strongswan_cert_win_002.png" width=500>
<img src="../../../../img/VPN/strongswan_cert_win_003.png" width=500>

<br />

다음으로는 윈도우 키를 누르고 VPN을 검색하여 VPN 설정에 들어간다.

VPN 연결 추가를 누른 뒤 VPN 공급자는 Windows(기본 제공), 연결 이름은 아무거나, 서버 이름 또는 주소엔 IP나 도메인, VPN 종류는 IKEv2, 로그인 정보 입력은 사용자 이름 및 암호, 사용자 이름과 암호는 ipsec.secrets에 기입한 아이디와 비밀번호를 각각 기입하여 준다.

다시 윈도우 키를 눌러 Powershell을 검색, 관리자 권한으로 연 다음 아래와 같이 입력한다. 

VPN_Name 부분에 앞서 추가한 VPN의 연결 이름을 적으면 된다.

```powershell
Set-VpnConnectionIPsecConfiguration -ConnectionName "VPN_Name" `
   -AuthenticationTransformConstants SHA256128 -CipherTransformConstants AES128 `
   -DHGroup Group14 -EncryptionMethod AES128 -IntegrityCheckMethod SHA256 -PFSgroup None -Force
```

위와 같이 하는 이유는 윈도우가 취약한 IPsec 매개 변수를 사용하여 연결 시도를 하니 서버에서 연결을 거부 당하기 때문이다. 참고로 PFS(Perfect Forward Secrecy)는 DHE(Diffie-Hellman Ephermeral) 알고리즘의 특성을 살려 주기적으로 대칭키를 변경하기에 서버 측에 PFS 설정을 하지 않은 앞선 포스트를 따라서 만들었다면 반드시 None으로 설정해야 한다. 그렇지 않으면 10분이 안되는 주기로 연결이 끊기는 현상을 맛보게 된다. (본인도 한 동안 애먹은 적이있다. 네트워크 보안 알고리즘 같은 건 배우지 않았는데 당시에 초심적 마음으로 이것저것 조합해서 따라하다보니 찾기까지 많은 시간이 걸렸었다. 처음엔 서버 dpd의 문제나 내 잘못인 줄 알고 며칠을 허비했는데, 그래도 안되서 하지 않은 것을 소거해보니 정확히 이해하지 못하는 기술들이 있어 그 특징 많이 배우게 됐다... 울어야 할지 웃어야 할지)


### iOS

iOS 기기에 ca-cert.cer를 받으면 Profile 등록이 뜬다. 설정으로 가서 과정에 따라 설치하고 VPN으로 진입하여 추가한다. 추가할 떄 유형으로는 IKEv2, 설명엔 아무 이름, 서버랑 원격 ID에는 ip주소(도메인은 해보지 않아서 정확히 모르겠다 아마 대동소이할 듯) 사용자 인증은 사용자 이름, 사용자 이름과 비밀번호에 ipsec.secrets에 기입한 아이디와 비밀번호를 각각 기입하여 주면 끝이다.