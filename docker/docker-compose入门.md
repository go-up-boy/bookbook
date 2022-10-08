# 基础知识

## 常用命令

1. docker-compose -h
2. docker-compose up
    * 启动所有 docker-compose 服务
3. docker-compose up -d
    * 启动所有 docker-compose服务并且后台运行
4. docker-compose down
    * 停止并且删除容器、网络、卷、镜像
5. docker-compose exec
    * yml里面的服务id
    * 进入容器实例内部
    * docker-compose exec docker-compose.yml 文件中写的服务id /bin/bash
6. docker-compose ps
    * 展示当前编排过运行中的所有容器
7. docker-compose top
    * 展示当前编排过容器进程
8. docker-compose logs yml id
    * 查看容器输出日志
9. docker-compose config
    * 检查配置
10. docker-compose config -q
    * 检查配置，有问题输出
11. docker-compose restart
    * 重启服务
12. docker-compose start
    * 启动
13. docker-compose stop
    * 停止

## 模板指令

# 开始编排
## 1. Redis + Mysql + PHP + Nginx






参考连接： https://blog.csdn.net/m0_46090675/article/details/121873943