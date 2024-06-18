# 송신자 프로그램 (sender.c)

## 개요
송신자 프로그램은 UDP를 통해 파일을 수신자에게 전송합니다. 이 프로그램은 신뢰할 수 있는 데이터 전송(Reliable Data Transfer, RDT) 프로토콜을 구현하며, 패킷 손실과 중복 패킷을 처리합니다. 송신자는 stop-and-wait 모델을 따르며, ACK를 기다린 후 다음 패킷을 전송합니다.

## 패킷 구조
- **type**: 패킷 타입 (DATA, ACK, EOT)
- **seqNum**: 시퀀스 번호
- **ackNum**: ACK 번호
- **length**: 데이터 길이 (0-1000 바이트)
- **data**: 실제 데이터 (최대 1000 바이트)

## 요구사항
- C 컴파일러 (예: `gcc`)

## 컴파일 방법
송신자 프로그램을 컴파일하려면 다음 명령어를 사용합니다:

```bash
gcc -o sender sender.c

## 실행 방법
./sender <sender_port> <receiver_ip> <receiver_port> <timeout_interval> <filename> <drop_prob>
- sender_port: 송신자 포트 번호
- receiver_ip: 수신자 IP 주소
- receiver_port: 수신자 포트 번호
- timeout_interval: 타임아웃 간격 (초)
- filename: 전송할 파일 이름
- drop_prob: ACK 패킷 폐기 확률 (0.0 ~ 1.0)
