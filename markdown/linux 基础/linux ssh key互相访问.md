# linux ssh key 互相访问

##  多台服务器互相无密码访问 

### 1 在每台服务器上执行 `ssh-keygen -t rsa `生成密钥对：

> ssh-keygen -t rsa

### 2 在每台服务器上生成密钥对后，将公钥复制到需要无密码访问的服务器上

> ssh-copy-id -i  ~/.ssh/id_rsa.pub [root@192.168.15.241](mailto:root@192.168.15.241)

若端口不是默认的22端口`ssh-copy-id -i ~/.ssh/id_rsa.pub -p 22 user@server`

