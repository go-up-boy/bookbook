# docker 安装 portainer

* 环境 windows wsl2
* 版本 portainer 2.15


1. 拉取最新版本 
```shell
docker pull portainer/portainer-ce:latest
```
2. 运行容器
    * 这里要注意 映射的套接字，根据自己环境来，我的是Wsl2路径是这样，linux默认在 /var/run/docker.sock
```shell
docker run -d -p 9000:9000 --name portainer_agent --restart=always -v \\wsl.localhost\Ubuntu-18.04\run\docker.sock:/var/run/docker.sock:/var/run/docker.sock -v C:\Users\z\portainer\data:/data portainer/portainer-ce:latest
```

3. 登录 127.0.0.1:9000 首次登录填写密码