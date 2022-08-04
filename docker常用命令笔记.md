```
# 编译镜像：
sudo docker build -t <image_name:image_version> .
# 运行容器：
sudo docker run -d -it <image_name:image_version> /bin/bash 
# 进入容器：
sudo docker exec -it <your container id, 4位即可> /bin/bash
# 查看所有容器
sudo docker container ls -a
# 查看正在运行的容器
sudo docker container ls
# 查看所有镜像
sudo docker image ls -a
# 停止容器
sudo docker stop <container-id>
# 删除指定容器
sudo docker rm <container-id>
# 删除指定镜像(镜像必须要在它关联的容器被删除之后才能删除)
sudo docker image rm <image_name:image_version>
# 删除所有容器
sudo docker container prune -a
# 删除所有镜像
sudo docker image prune -a
# 进入运行中的容器
sudo docker exec -it <container-id> /bin/bash
```