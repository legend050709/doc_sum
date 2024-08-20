```table-of-contents
```
# dockerfile

## dockerfile编辑
## 基于dockerfile编译镜像
```bash
- mkdir docker_build_dir
- cd docker_build_dir
- edit dockerfile
- docker build -f ./dockerfile -t dns-compiling-env-100g .
- docker image ls
```

## docker镜像操作
### 查看docker镜像
```bash
docker image ls
```
### 操作docker实例
```bash
// 执行docker镜像（即启动一个docker实例）
docker run --cap-add=ALL -v $(pwd):/dns  --name $(whoami) -dit dns-compiling-env-100g /bin/bash

docker run --cap-add=ALL -v $(pwd):/dns [-v /lib/modules:/lib/modules] --name $(whoami) -dit dns-compiling-env-100g /bin/bash
	// 宿主机的当前目录，对应docker中的dns目录；
	// -dit: 镜像名称
	// --name: docker实例的名称;
	// /bin/bash： 进入docker后执行的bash/脚本;
```
### 进入docker实例
```bash
	docker exec -it $(whoami) /bin/bash
```

### 查看docker信息
```bash
 // 查看信息
docker info
docker ps -a

```

### 启停docker进程
```bash
// 停止docker实例
docker stop CONTAINER-ID

// 启动docker实例
docker start CONTAINER-ID

// 删除docker实例
docker rm CONTAINER-ID
```

### 上传/下载镜像
#### 上传到仓库
```bash
sudo docker image ls
sudo docker tag xxx registry-internal.corp.mynet.com/dns/xxx
sudo docker login -u xxx  http://registry-internal.corp.mynet.com
sudo docker push registry-internal.corp.mynet.com/dns/xxx
//Note: Before uploading to the repository, make sure you have administrative rights to the repository

sudo docker run --cap-add=ALL -v $(pwd):/dns [-v /lib/modules:/lib/modules] --name $(whoami)_test -dit registry-internal.corp.kuaishou.com/dns/dns-compiling-env-dpdk20.11.1  bash

sudo docker exec -it $(whoami)_test /bin/bash
```

#### 本地上传下载
```bash
docker image ls

// 使用docker save命令将镜像保存为tar文件。例如，以下命令将名为myimage的镜像保存为myimage.tar文件
docker save myimage > myimage.tar

// 可以使用docker load命令将tar文件加载为Docker镜像
docker load < myimage.tar
```

# 参考
```bash

```