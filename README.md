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
# 複製範本
重啟svn容器
```
./start.sh
```
將相關設定檔，複製到volume中
```
./enter.sh

  cp /etc/subversion/* /tmp/svn/svn_config
  cp /etc/apache2/conf.d/* /tmp/svn/apache2_config
  exit
```
此時相關設定檔已經同步到HOST主機中

# 修改腳本
更改start.sh
```
cat << EOF >start.sh
#!/bin/bash
docker stop svn-test
docker rm svn-test
docker run --add-host='ldap.alle:59.125.14.2' --restart always --name svn-test -d -p 3690:3690 -p 8080:80 \\
    -v /docker/volumes/svn/apache2_config:/etc/apache2/conf.d \\
    -v /docker/volumes/svn/svn_repo:/home/svn \\
    -v /docker/volumes/svn/svn_config:/etc/subversion \\
    172.16.9.161:5000/library/svn-server
EOF
```

# 更改volume設定後，重啟容器
```
./start.sh
```

# 進入容器，創建倉庫
```
./enter.sh
  mkdir -p /home/svn/myrepo
  svnadmin create /home/svn/myrepo
  exit
```

# 設定SVN權限
```
cat << EOF >svn_config/subversion-access-control
[groups]
[/]
* = rw
admin = rw
EOF

htpasswd -b svn_config/passwd admin admin
chmod a+w svn_config/*
```

#設定APACHE
```
cat << EOF > apache2_config/dav_svn.conf
LoadModule dav_svn_module /usr/lib/apache2/mod_dav_svn.so
LoadModule authz_svn_module /usr/lib/apache2/mod_authz_svn.so

<Location />
     DAV svn
     SVNParentPath /home/svn
     SVNListParentPath On
     AuthBasicProvider ldap file
     AuthType Basic
     AuthName "Subversion Repository"
     AuthUserFile /etc/subversion/passwd
     #AuthzLDAPAuthoritative on
     AuthLDAPURL "ldap://ldap.alle:389/DC=developer,DC=alle,DC=com?cn?sub?(objectClass=*)" NONE
     AuthzSVNAccessFile /etc/subversion/subversion-access-control
     Require valid-user
</Location>
EOF

chmod a+w apache2_config/*
```

# 完成所有設定後，重啟容器
```
./start.sh
```

# 查看成果
http://172.16.9.116:8080/
```
svn co http://172.16.9.116:8080/myrepo myrepo
```
