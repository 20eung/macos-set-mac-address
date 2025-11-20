# MacOS에서 외장 랜카드의 MAC 주소 영구 설정

## 1. 랜카드 식별 및 MAC 주소 확인

```bash
networksetup -listallhardwareports
```

```output
Hardware Port: Dell Universal Dock D6000
Device: en9
Ethernet Address: 9c:eb:e8:a8:dc:e9
```

---

## 2. MAC 주소 수동 설정 명령

```bash
sudo ifconfig [인터페이스이름] ether [원하는 MAC 주소]
```
예제
```bash
sudo ifconfig en9 ether f8:e4:3b:04:23:20
```


---

## 3. Launch Daemon 설정 (영구 유지)

Launch Daemon은 시스템 부팅 시 또는 특정 조건에서 스크립트를 실행하도록 macOS에 지시하는 메커니즘

### A. 설정 스크립트 작성

#### 1. 터미널에서 다음 명령어로 MAC 주소 설정 스크립트 파일을 생성하고 편집

```bash
sudo vi /usr/local/bin/set_mac_address.sh
```

#### 2. 편집기에 다음 내용을 입력하고 저장. [인터페이스 이름]과 [원하는 MAC 주소]를 실제 값으로 교체

```bash
#!/bin/bash

# 설정 변수
INTERFACE="[인터페이스 이름]"
TARGET_MAC="[원하는 MAC 주소]"

# 무한 루프 시작 (시스템이 켜져 있는 동안 계속 감시)
while true; do
    # 1. 인터페이스가 존재하는지 확인
    if ifconfig "$INTERFACE" >/dev/null 2>&1; then

        # 2. 현재 MAC 주소 확인
        CURRENT_MAC=$(ifconfig "$INTERFACE" | awk '/ether/ {print $2}')

        # 3. MAC 주소가 목표치와 다르면 변경 시도
        if [ "$CURRENT_MAC" != "$TARGET_MAC" ]; then
            /sbin/ifconfig "$INTERFACE" ether "$TARGET_MAC"
            echo "$(date): $INTERFACE MAC address changed to $TARGET_MAC" >> /tmp/set_mac_address.log
        fi
    fi

    # 5초 대기 후 다시 확인 (CPU 점유율 방지)
    sleep 5
done
```

#### 3. 스크립트에 실행 권한을 부여

```bash
sudo chmod +x /usr/local/bin/set_mac_address.sh
```

### B. Launch Daemon 파일 작성

#### 1. Launch Daemon 설정 파일(Plist 파일)을 생성

```bash
sudo vi /Library/LaunchDaemons/com.local.setmac.plist
```

#### 2. 편집기에 다음 내용을 입력하고 저장

```xlm
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.local.setmac</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/set_mac_address.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardErrorPath</key>
    <string>/tmp/set_mac_address.err</string>
    <key>StandardOutPath</key>
    <string>/tmp/set_mac_address.out</string>
</dict>
</plist>
```

### C. Launch Daemon 로드

#### 1. 생성한 Plist 파일의 소유권을 ```root```로 변경, 권한을 ```644```로 변경 (소유자만 쓰기 가능, 나머지는 읽기만 가능)

```bash
sudo chown root:wheel /Library/LaunchDaemons/com.local.setmac.plist
sudo chmod 644 /Library/LaunchDaemons/com.local.setmac.plist
```

#### 2. 서비스 등록 (Bootstrap 방식 사용)

```bash
sudo launchctl bootstrap system /Library/LaunchDaemons/com.local.setmac.plist
```

---

## 4. 확인

### A. **현재 상태 확인**: 터미널에서 프로세스가 돌고 있는지 확인

```bash
ps aux | grep set_mac_address.sh
```

### B. 실전 테스트:

* USB 허브를 **연결 해제**
* USB 허브를 **다시 연결**
* 약 5~10초 기다린 후, ```ifconfig en9``` 명령어로 MAC 주소가 바뀌었는지 확인
```bash
ifconfig en9 | grep ether
```
* 로그 파일을 확인하여 변경 기록 점검
```bash
cat /tmp/set_mac_address.log
```


### C. **시스템을 재부팅**

### D. 재부팅 후 다시 MAC 주소를 확인하여 설정한 주소가 유지되는지 확인
