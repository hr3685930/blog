# openVPN安装

# 初始化服务器

```
export SERVER_ADDR=47.97.185.64
docker run -v /data/openvpn:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://$SERVER_ADDR:1194
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
docker run -v /data/openvpn:/etc/openvpn --rm -p 1194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn
```

# 注册用户
```
export CLIENT_NAME=guojia
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full $CLIENT_NAME nopass
docker run -v /data/openvpn:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient $CLIENT_NAME > /data/ovpn/$CLIENT_NAME.ovpn
```

# 查看用户列表
```
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn ovpn_listclients
```

# 撤销用户访问
```
export CLIENT_NAME=guojia
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn easyrsa revoke $CLIENT_NAME
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn easyrsa gen-crl
cp /data/openvpn/pki/crl.pem /data/openvpn/crl.pem
```
PS. 第一次要重新启动Docker，不然不生效

https://tunnelblick.net/downloads.html

https://hub.docker.com/r/kylemanna/openvpn/

https://github.com/kylemanna/docker-openvpn