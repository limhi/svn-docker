[![Docker Image](https://img.shields.io/badge/docker%20image-available-green.svg)](http://172.16.9.161:5000/v2/_catalog)

# Description
forked from elleFlorio/svn-docker. 加入apache2-ldap支持，刪除svnadmin的php工具，

# private docker
設定內部docker倉庫
```
cat << EOF > /etc/docker/daemon.json
{
  "insecure-registries": [
    "172.16.9.161:5000"
    ]
}
EOF

systemctl daemon-reload
systemctl restart docker
```

建立image，上傳到內部倉庫
```
docker build -t 172.16.9.161:5000/library/svn-server:latest .
docker push 172.16.9.161:5000/library/svn-server:latest
```
查看http://172.16.9.161:5000/v2/_catalog

# 建立環境
base dir: /docker/volumes/svn
```
mkdir -p /docker/volumes/svn
cd /docker/volumes/svn
mkdir svn_repo svn_config apache2_config
sudo chmod -R a+w *
```

# 建立腳本
generate enter.sh
```
cat << EOF >enter.sh
#!/bin/bash
docker exec -it svn-test /bin/sh
EOF

chmod +x *.sh
```

generate temp start.sh
```
cat << EOF >start.sh
#!/bin/bash
docker stop svn-test
docker rm svn-test
docker run --restart always --name svn-test -d -p 3690:3690 -p 8080:80 \\
       -v /docker/volumes/svn:/tmp/svn \\
       172.16.9.161:5000/library/svn-server
EOF

chmod +x *.sh
```
