# 송신자 프로그램

import socket
import os
import random
import time

# 패킷 타입 및 플래그 상수 정의
DATA = 0
ACK = 1
SYN = 2
FIN = 3

NONE = 0
SYN_FLAG = 1
FIN_FLAG = 2

class TCPSender:
    def __init__(self, sender_port, receiver_ip, receiver_port, timeout, filename, drop_prob):
        self.sender_port = sender_port
        self.receiver_address = (receiver_ip, receiver_port)
        self.timeout = timeout
        self.filename = filename
        self.drop_prob = drop_prob

        self.sender_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.sender_socket.bind(('', sender_port))
        self.sender_socket.settimeout(timeout)

        self.cwnd = 1
        self.ssthresh = 64
        self.base = 0
        self.next_seq_num = 0
        self.duplicate_acks = 0
        self.start_time = time.time()
    
    def log_event(self, event, packet=None):
        timestamp = time.time() - self.start_time
        if packet:
            print(f"[{timestamp:.2f}] {event} - SeqNum: {packet.seq_num}, AckNum: {packet.ack_num}, CWND: {self.cwnd}")
        else:
            print(f"[{timestamp:.2f}] {event}")

    def handshake(self):
        # 연결 설정: 3-way 핸드셰이크
        syn_pkt = Packet(DATA, SYN_FLAG, 0)
        self.sender_socket.sendto(syn_pkt.to_bytes(), self.receiver_address)
        self.log_event("Sent SYN", syn_pkt)
        
        try:
            response, _ = self.sender_socket.recvfrom(1024)
            syn_ack_pkt, _ = Packet.from_bytes(response)
            if syn_ack_pkt.flag == SYN_FLAG and syn_ack_pkt.ack_num == 1:
                self.log_event("Received SYN-ACK", syn_ack_pkt)
                ack_pkt = Packet(ACK, NONE, 1, syn_ack_pkt.seq_num + 1)
                self.sender_socket.sendto(ack_pkt.to_bytes(), self.receiver_address)
                self.log_event("Sent ACK", ack_pkt)
                return True
        except socket.timeout:
            self.log_event("Handshake failed due to timeout")
        return False

    def send_file(self):
        # 파일을 작은 청크로 나누기
        with open(self.filename, 'rb') as file:
            chunks = [file.read(1000) for _ in range(0, os.path.getsize(self.filename), 1000)]
        
        while self.base < len(chunks):
            while self.next_seq_num < self.base + self.cwnd and self.next_seq_num < len(chunks):
                data_pkt = Packet(DATA, NONE, self.next_seq_num, 0, chunks[self.next_seq_num].decode())
                self.sender_socket.sendto(data_pkt.to_bytes(), self.receiver_address)
                self.log_event("Sent DATA", data_pkt)
                self.next_seq_num += 1

            self.sender_socket.settimeout(self.timeout)
            try:
                ack_data, _ = self.sender_socket.recvfrom(1024)
                ack_pkt, _ = Packet.from_bytes(ack_data)
                if ack_pkt.type == ACK:
                    self.log_event("Received ACK", ack_pkt)
                    if ack_pkt.ack_num > self.base:
                        self.base = ack_pkt.ack_num
                        self.duplicate_acks = 0
                        if self.cwnd < self.ssthresh:
                            self.cwnd *= 2
                        else:
                            self.cwnd += 1
                    elif ack_pkt.ack_num == self.base:
                        self.duplicate_acks += 1
                        if self.duplicate_acks == 3:
                            self.ssthresh = max(self.cwnd // 2, 1)
                            self.cwnd = self.ssthresh + 3
                            self.sender_socket.sendto(chunks[self.base].to_bytes(), self.receiver_address)
                            self.log_event("Fast Retransmit", ack_pkt)
            except socket.timeout:
                self.log_event("Timeout, retransmitting")
                self.ssthresh = max(self.cwnd // 2, 1)
                self.cwnd = 1
                self.next_seq_num = self.base
                self.duplicate_acks = 0
    
    def teardown(self):
        # 연결 종료: 4-way 핸드셰이크
        fin_pkt = Packet(DATA, FIN_FLAG, self.next_seq_num)
        self.sender_socket.sendto(fin_pkt.to_bytes(), self.receiver_address)
        self.log_event("Sent FIN", fin_pkt)

        try:
            response, _ = self.sender_socket.recvfrom(1024)
            fin_ack_pkt, _ = Packet.from_bytes(response)
            if fin_ack_pkt.flag == FIN_FLAG:
                self.log_event("Received FIN-ACK", fin_ack_pkt)
                ack_pkt = Packet(ACK, NONE, fin_ack_pkt.seq_num + 1)
                self.sender_socket.sendto(ack_pkt.to_bytes(), self.receiver_address)
                self.log_event("Sent ACK", ack_pkt)
        except socket.timeout:
            self.log_event("Teardown failed due to timeout")

if __name__ == '__main__':
    import sys
    if len(sys.argv) != 7:
        print("Usage: sender.py <sender_port> <receiver_ip> <receiver_port> <timeout> <filename> <drop_prob>")
        sys.exit(1)

    sender_port = int(sys.argv[1])
    receiver_ip = sys.argv[2]
    receiver_port = int(sys.argv[3])
    timeout = float(sys.argv[4])
    filename = sys.argv[5]
    drop_prob = float(sys.argv[6])

    sender = TCPSender(sender_port, receiver_ip, receiver_port, timeout, filename, drop_prob)
    if sender.handshake():
        sender.send_file()
        sender.teardown()
