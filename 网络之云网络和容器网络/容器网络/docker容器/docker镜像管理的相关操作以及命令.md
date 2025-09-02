```table-of-contents
```


# docker镜像操作
## 查看docker镜像
```bash
docker image ls // docker镜像
docker ps -a // docker 进程实例
```


## 生成docker镜像
### 基于dockerfile生成镜像image
```bash
- mkdir docker_build_dir
- cd docker_build_dir
- edit dockerfile
- docker build -f ./dockerfile -t dns-compiling-env-100g .
dns-compiling-env-100g: docker image 名称
-f ./dockerfile： 指定dockerfile文件

- docker image ls # 生成的镜像查看
```

### 基于运行的容器生成镜像
在 Docker 中，从正在运行的容器生成新的镜像是一个常见的操作，可以用于保存容器的当前状态或将其用作新镜像的基础。

```bash
# 生成
docker commit [OPTIONS] CONTAINER IMAGE[:TAG]
- `CONTAINER` 是要生成镜像的容器的 ID 或名称。
- `IMAGE[:TAG]` 是新镜像的名称和可选标签。

比如：docker commit -m "Added new features" -a "Your Name" my_container my_new_image

# 验证
docker images

```
## 上传/下载镜像
### 上传到仓库
```bash
sudo docker image ls
sudo docker tag xxx registry-internal.corp.mynet.com/dns/xxx
sudo docker login -u xxx  http://registry-internal.corp.mynet.com
sudo docker push registry-internal.corp.mynet.com/dns/xxx
//Note: Before uploading to the repository, make sure you have administrative rights to the repository

sudo docker run --cap-add=ALL -v $(pwd):/dns [-v /lib/modules:/lib/modules] --name $(whoami)_test -dit registry-internal.corp.kuaishou.com/dns/dns-compiling-env-dpdk20.11.1  bash

sudo docker exec -it $(whoami)_test /bin/bash
```

### 本地上传下载
```bash
docker image ls

// 使用docker save命令将镜像保存为tar文件。例如，以下命令将名为myimage的镜像保存为myimage.tar文件
docker save myimage > myimage.tar

// 可以使用docker load命令将tar文件加载为Docker镜像
docker load < myimage.tar
```

# 容器操作
## 执行docker实例(容器)
执行docker实例(容器) 即 启动一个后台进程。

```bash
// 执行docker镜像（即启动一个docker实例）
docker run --cap-add=ALL -v $(pwd):/dns  --name $(whoami) -dit dns-compiling-env-100g /bin/bash

docker run --cap-add=ALL -v $(pwd):/dns [-v /lib/modules:/lib/modules] --name $(whoami) -dit dns-compiling-env-100g /bin/bash
	// 宿主机的当前目录，对应docker中的dns目录；
	// -dit: 镜像名称
	// --name: docker实例的名称;
	// /bin/bash： 进入docker后执行的bash/脚本;
```

## 进入docker实例(容器)
```bash
	docker exec -it $(whoami) /bin/bash
```

## 查看docker信息
```bash
 // 查看信息
docker info
docker ps -a // docker 进程实例

```

## 启停docker进程(容器)
```bash
// 停止docker实例
docker stop CONTAINER-ID

// 启动docker实例
docker start CONTAINER-ID

// 删除docker实例
docker rm CONTAINER-ID
```



# 参考
```bash

```