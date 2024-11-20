```table-of-contents
```
# udp测试
## 不同大小载荷的测试

### 测试原理

**（1）测试目的**：
为了测试网关的 inbound、outbound方向对于各个大小的数据包的转发是否正常，尤其是网关存在隧道封装的情况（比如：vxlan的封装、ipip隧道的封装）等。

**（2）测试原理**：
client 给 server端发送不同大小载荷的UDP数据包，同时server端会回复给client同样大小的响应包。
如果server端在1s内没有回复响应，client则认为超时，说明对于这个大小的数据包的转发存在异常。
### python实现
#### server端
```python
import socket
import sys

HOST = sys.argv[1]
PORT = int(sys.argv[2])

af = socket.AF_INET
if ':' in HOST:
    af = socket.AF_INET6
with socket.socket(af, socket.SOCK_DGRAM) as s:
    s.bind((HOST, PORT))
    while True:
        bytesAddressPair = s.recvfrom(4096)
        data = bytesAddressPair[0]
        addr = bytesAddressPair[1]
        print(f"received {addr} %d" % len(data))
        s.sendto(bytes(len(data)), addr)

```
#### client端
```python
import socket
import sys

HOST = sys.argv[1]
PORT = int(sys.argv[2])
serverAddressPort = (HOST, PORT)

af = socket.AF_INET
if ':' in HOST:
    af = socket.AF_INET6
for l in range(1,1501):
    with socket.socket(af, socket.SOCK_DGRAM) as s:
        s.settimeout(1)
        s.sendto(bytes(l), serverAddressPort)
        data = s.recvfrom(4096)
        if len(data[0]) != l:
            print("failed,send %d, received %d"%(l, len(data[0])))
        else:
            print(f"ok {l}")
        s.close()


```
## 不同五元组的测试
### 测试原理

**（1）测试目的**：
网络存在==偶发超时、丢包==的情况下，不确定是否是特定的五元组存在丢包。因为不同的五元组走的网路的路径是不一样的。最终，为了查找存在问题的中间网络设备。

**（2）测试原理**：
client 给 server端发送 `SIP:DIP:DPort` 固定不变，sport变化的数据包，server 收到包之后，给client发送响应包；如果1s内没有收到响应包，则说明这个特定的五元组的流量在inbound 或者 outbound 方向存在丢包。
### python实现
#### server端
```python
import socket
import sys

HOST = sys.argv[1]
PORT = int(sys.argv[2])

af = socket.AF_INET
if ':' in HOST:
    af = socket.AF_INET6
with socket.socket(af, socket.SOCK_DGRAM) as s:
    s.bind((HOST, PORT))
    while True:
        bytesAddressPair = s.recvfrom(4096)
        data = bytesAddressPair[0]
        addr = bytesAddressPair[1]
        print(f"received {addr} %d" % len(data))
        s.sendto(bytes(len(data)), addr)
```
#### client端
```python
import socket
import sys

HOST = sys.argv[1]
PORT = int(sys.argv[2])
serverAddressPort = (HOST, PORT)

af = socket.AF_INET
if ':' in HOST:
    af = socket.AF_INET6

for l in range(1024, 65535):
    try:
        with socket.socket(af, socket.SOCK_DGRAM) as s:
            s.settimeout(1)
            s.bind(('', l))
            s.sendto(bytes(64), serverAddressPort)
            client_port = s.getsockname()[1]  # 在发送数据后获取客户端端口号
            data = s.recvfrom(4096)
            if len(data[0]) != 64:
                print("failed, send %d, received %d" % (l, len(data[0])))
            else:
                print(f"ok {l}")
    except socket.timeout:
        print(f"Timeout error, Client Port: {client_port}")
    except Exception as e:
        print(f"Error: {e}, Client Port: {client_port}")
```
# tcp测试
## 不同大小载荷的测试
### python实现
#### server端
```python
import socket
import sys

HOST = ""
PORT = int(sys.argv[1])

with socket.socket(socket.AF_INET6, socket.SOCK_STREAM) as s:
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
    s.bind((HOST, PORT))
    s.listen()
    while True:
        conn, addr = s.accept()
        try:
            conn.settimeout(1)
            while True:
                data = conn.recv(4096)
                if len(data) == 0:
                    break
                print(f"received {addr} %d" % len(data))
                conn.sendall(data)
        except:
            print(f"failed {addr}")
        finally:
            conn.close()
```

#### client端
```python
import socket
import sys
import time

HOST = sys.argv[1]
PORT = int(sys.argv[2])

af = socket.AF_INET
if ':' in HOST:
    af = socket.AF_INET6
for l in range(1,1501):
    with socket.socket(af, socket.SOCK_STREAM) as s:
        s.settimeout(1)
        s.connect((HOST, PORT))
        s.sendall(bytes(l))
        rl = 0
        while True:
            data = s.recv(4096)
            if len(data) == 0:
                break
            rl += len(data)
            if rl == l:
                break
        if rl != l:
            print("failed,send %d, received %d"%(l, len(data)))
        else:
            print(f"ok {l}")
        s.close()

```
## 不同五元组的测试
### python实现
#### server端
```python

```
#### client端
```python

```

# 参考
```bash

```