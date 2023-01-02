由于我们的方向实践课项目( **volatile_reborn** )基于上学期的软工三项目( **volatile** ), 后者已经初步使用了Swarm集群了. 因此我在这里演示一下这个集群演进的过程.

# **Swarm实战: volatile**

我们要将前端服务`volatile_frontend_svc`(容器监听80端口)部署到集群，对外暴露集群的81端口

## **Server Config**

| 主机名          | 主机ip | 角色                 |
| --------------- | ------ | -------------------- |
| lyk阿里云服务器 | **     | master，CICD工作节点 |
| lyk华为云服务器 | **     | master (目前已失效)  |
| lyk腾讯云服务器 | **     | master               |

3台Master， 3台Worker（ Master也同时作为Worker，因此实际上只有三台主机 ）

## **Prerequists**

前置准备：所有节点必须打开：

- UDP/4789： 绑定到VTE

- TCP/2377： Swarm的集群管理默认使用2377端口

- TCP/7946, UDP/7946: Swarm的节点发现使用7946端口

- TCP81: 暴露所有运行前端server的node的的81端口

## **步骤**

1. 先在阿里云主机上创建第一个master节点：

1. docker  swarm init --advertise-addr **:2377
   1. 填该master节点的公网ip

1. 生成令节点作为master加入集群的令牌：

1. docker swarm join-token manager
   1. 该命令的输出即为令牌

1. 在其他节点上使用上述令牌，使其作为master加入该swarm集群。 成功后执行下述命令，查看集群中的节点：

1. docker node ls 

1. 在任意master上创建overlay网络，名为`volatile`:

1. docker network create -d overlay volatile
   1. 一定要先创建网络, 否则其他节点无法加入该网络

1. 在master上基于镜像创建新服务`volatile_frontend_svc`，并使用网络`volatile`:

1. docker service create --name volatile_frontend_svc \ --network volatile \ -p 81:80 \ --replicas 3 \ lyklove/volatile_frontend:latest
   1. 这里设置服务实例数为3
   2. 我们将集群的81端口映射到了容器的80端口。 因此访问集群的任意节点的81端口的流量最终都会被转发到运行了该服务副本的节点

1. （后续）滚动更新:

1. docker service update --image lyklove/volatile_frontend:new --update-parallelism 2  --update-delay 1s volatile_frontend_svc

- 基于新镜像`lyklove/volatile_frontend:new`更新服务，并在其他节点上也进行服务更新

- 该命令可以在任意拥有该新镜像的master节点上执行

## **集成CICD**

可以发现，CICD只需要将服务打包成镜像，然后利用该镜像滚动更新就行了

- Jenkins脚本`jenkinsfile.groovy`:

  ```Plain
  // 根据代码构建新镜像
  stage("update service by built image"){
          sh "docker service update --image ${IMAGE_TO_RUN} --update-parallelism 2  --update-delay 2s ${SERVICE_NAME}"
      }
  ```

  - 注意, jenkinsfile里只写了`docker service update`, 而没有create. 因此, 需要先手动在server上create service, 后续cicd时才能够update

## **集群使用**

通过`[host-ip]:81`访问前端

其中`host-ip`可以是集群中任意节点的ip

- # Swarm实战: volatile

  我们要将前端服务`volatile_frontend_svc`(容器监听80端口)部署到集群，对外暴露集群的81端口

  ## Server Config

  | 主机名          | 主机ip | 角色                 |
  | --------------- | ------ | -------------------- |
  | lyk阿里云服务器 | **     | master，CICD工作节点 |
  | lyk华为云服务器 | **     | master               |
  | lyk腾讯云服务器 | **     | master               |

  3台Master， 3台Worker（ Master也同时作为Worker，因此实际上只有三台主机 ）

  ## Prerequists

  前置准备：所有节点必须打开：

  - UDP/4789： 绑定到VTE
  - TCP/2377： Swarm的集群管理默认使用2377端口
  - TCP/7946, UDP/7946: Swarm的节点发现使用7946端口
  - TCP81: 暴露所有运行前端server的node的的81端口

  ## 步骤

  1. 先在阿里云主机上创建第一个master节点：

     ```
     docker  swarm init --advertise-addr **:2377
     ```

     * 填该master节点的公网ip

  2. 生成令节点作为master加入集群的令牌：

     ```sh
     docker swarm join-token manager
     ```

     * 该命令的输出即为令牌

  3. 在其他节点上使用上述令牌，使其作为master加入该swarm集群。 成功后执行下述命令，查看集群中的节点：

     ```sh
     docker node ls 
     ```

  4. 在任意master上创建overlay网络，名为`volatile`:

     ```
     docker network create -d overlay volatile
     ```

     * 一定要先创建网络, 否则其他节点无法加入该网络

  5. 在master上基于镜像创建新服务`volatile_frontend_svc`，并使用网络`volatile`:

     ```sh
     docker service create --name volatile_frontend_svc \
     --network volatile \
     -p 81:80 \
     --replicas 3 \
     lyklove/volatile_frontend:latest
     ```

     * 这里设置服务实例数为3
     * 我们将集群的81端口映射到了容器的80端口。 因此访问集群的任意节点的81端口的流量最终都会被转发到运行了该服务副本的节点

  6. （后续）滚动更新:

     ```shell
     docker service update --image lyklove/volatile_frontend:new --update-parallelism 2  --update-delay 1s volatile_frontend_svc
     ```

  - 基于新镜像`lyklove/volatile_frontend:new `更新服务，并在其他节点上也进行服务更新
  - 该命令可以在任意拥有该新镜像的master节点上执行

  ## 集成CICD

  可以发现，CICD只需要将服务打包成镜像，然后利用该镜像滚动更新就行了

  

  Jenkins脚本`jenkinsfile.groovy`:

  ```java
  // ...
  // 根据代码构建新镜像
  stage("update service by built image"){
          sh "docker service update --image ${IMAGE_TO_RUN} --update-parallelism 2  --update-delay 2s ${SERVICE_NAME}"
      }
  ```

  * 注意, jenkinsfile里只写了`docker service update`, 而没有create. 因此, 需要先手动在server上create service, 后续cicd时才能够update

  ## 集群使用

  通过`[host-ip]:81`访问前端

  其中`host-ip`可以是集群中任意节点的ip



# Swarm实战: volatile_reborn

## Server Config

由于华为云服务器过期了, 这次只有两个节点. 这也意味着只能有一个专业的worker, 否则会发生brain-split

| 主机名          | 主机ip | 角色                 |
| --------------- | ------ | -------------------- |
| lyk腾讯云服务器 | **     | master, CICD工作节点 |
| lyk阿里云服务器 | **     | master               |

1台Master， 2台Worker(算上Master)

## Steps

1. 照常配置服务器端口

2. 把之前volatile时期配置的华为云的node删掉. 由于server已经登陆不上了, 没法主动leave, 只能在其他节点上删除:

   ```sh
   docker node rm  k8s-master //k8s-master是华为云服务器的hostname
   ```

3. Create Token:

   ```
   docker  swarm init --advertise-addr 123.56.20.222:2377
   ```

4. Let other nodes join the swarm with this token

5. Create overlay network on  any master, 名为`volatile_reborn`:

   ```
   docker network create -d overlay volatile_reborn
   ```

6. Create frontend service on manager:

   ```sh
   docker service create \
   --name frontend_volatile_reborn_svc \
   --network volatile_reborn -p 81:80 \
   --replicas 2 lyklove/frontend_volatile_reborn:latest-linux 
   ```

7. Create backend  service on manager:

   ```sh
   docker service create \
   --name backend_volatile_reborn_svc \
   --network volatile_reborn -p 8000:8000 \
   --replicas 1 lyklove/backend_volatile_reborn:latest-linux 
   ```

8. Create eureka service on manager:

   ```
   docker service create \
   --name backend_eureka_volatile_reborn_svc \
   --network volatile_reborn -p 8001:8001 \
   --replicas 1 lyklove/backend_eureka_volatile_reborn:latest-linux 
   ```

9. 后续滚动更新和集成cicd都和volatile类似

