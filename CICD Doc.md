# Build Notes

由于现在大四了, 组员们的服务器都过期了, 所以本项目的服务器资源真的不够.

* 前端一构建就卡死(卡在`npm build`), 由于前后端+Jenkins都部署在同一台Host, 服务器卡死了整个项目全都不能用了, 因此请**千万不要随便触发前端的构建**
* 后端构建一次大概五六分钟, 但也有卡死的可能....
* 算法端也是一构建就卡死

# Jenkins

* Jenkins以容器形式部署

## Visit

* Jenkins地址： http://124.222.135.47:8081/job/backend-volatile/

* 用户名： lyk

* 密码： 123456

## Execution Logic

前后端的执行逻辑都相同:

1. 从git仓库clone代码
2. cd进入工作目录
3. 预处理. 比如查看一下工作目录下是否有需要的依赖( maven, node ... ); 或者先构建一些build镜像时所需的文件:
   * 后端：我写了个shell脚本来构建
4. 直接把项目构建成镜像：
5. 登录Dockerhub
6. 将镜像push到Dockerhub
7. 根据镜像来运行容器或者Docker Swarm Service

# Jenkinsfile

 我们的Jenkins有2个Item, 前后端各一个, 因此有两份Jenkinsfile.

## Frontend

```groovy
node("slave1") {
    def workspace = pwd()

    def git_branch = 'master'

    // def git_repository = 'git@git.nju.edu.cn:191820133/frontend-volatile.git' //Gitlab
    def git_repository = 'git@github.com:VolatileReborn/Frontend-VolatileReborn.git' //Github

    def vm_ip = '124.222.135.47'
    def vm_port = '22'
    def vm_user = 'lyk'

    def __vm_project_place = "/usr/local/src"
    def __vm_target_place = "/usr/local/src/target/"

    def __PROJECT_NAME = 'volatile_reborn'
    def __PROJECT_TYPE = 'frontend'
    def __DOCKERHUB_ACCOUNT = 'lyklove'
    def __IMAGE_TAG = 'latest-linux'

    def PUBLIC_PORT = '81'
    def CONTAINER_PORT = '80' // 80 for VUE

    def ORIGINAL_IMAGE_NAME = __PROJECT_TYPE + '_' + __PROJECT_NAME //frontend_volatile_reborn
    def IMAGE_NAME_WITH_INITIAL_TAG = ORIGINAL_IMAGE_NAME + ':' + __IMAGE_TAG //frontend_volatile_reborn:latest
    def IMAGE_FULL_NAME = __DOCKERHUB_ACCOUNT + '/' + IMAGE_NAME_WITH_INITIAL_TAG // lyklove/frontend_volatile_reborn:latest

    def CONTAINER_NAME = ORIGINAL_IMAGE_NAME //frontend_volatile_reborn
    def SERVICE_NAME = CONTAINER_NAME + '_svc' //frontend_volatile_reborn_svc

    stage('clone from github into slave\'s workspace. Using branch: ' + "master") {
        echo "workspace: ${workspace}"
        git branch: "${git_branch}", url: "${git_repository}"
    }


    stage('cd to build context') {
        echo "the context now is:"
        // sh "ls -al"
        sh "cd ${workspace}"
        echo "cd to build context, now the context is:"
        // sh "ls -al"
    }
    
    //@Deprecated
    // stage('get version info'){
    //     sh 'node -v'
    //     sh 'npm -v'
    //     sh 'vue -V'
    // }

//     stage('build with npm') {

// //         sh 'npm config set registry http://registry.cnpmjs.org'
//         // sh 'npm install -g @vue/cli'
//         // sh 'npm install vue@next'
//         sh 'npm install --registry=https://registry.npm.taobao.org' //必须加代理, 不然很慢
//         sh 'npm list vue'
//         sh 'npm run build'
//         echo "build finish on ${vm_ip}"
//     }

//@DEBUG
//     stage('npm run serve'){
//
//         echo 'not using docker yet!!'
//         sh 'npm run serve'
//     }


    stage("build docker image"){
        //使用node-alpine
        def DOCKERFILE_PATH = './Dockerfile.node-alpine'
        
        sh "docker build -t ${IMAGE_FULL_NAME}  -f  ${DOCKERFILE_PATH} ."
//         sh "imageId=`docker images | grep #{IMAGE_NAME} | awk '{print $3}'`"
    }

    //using Swarm, no need to docker run
    // stage("run docker container"){
    //     // 一定要加-d, 否则docker run就会一直运行, 导致jenkins构建无法结束
    //     sh "docker container run  -d -p ${PUBLIC_PORT}:${CONTAINER_PORT} --rm --name   ${CONTAINER_NAME}  ${IMAGE_FULL_NAME}"
    // }

    //虽然会很卡, 但是docker swarm必须要求使用registry的镜像, 所以必须push到dockerhub
    stage("login to dockerhub and push"){
        withCredentials([usernamePassword(credentialsId: 'DOCKERHUB_KEY', passwordVariable: 'password', usernameVariable: 'username')]) {
            sh 'docker login -u $username -p $password'
        }
        sh "docker image push ${IMAGE_FULL_NAME}"
    }


    //Using docker service
    //需要先在服务器上手动创建该service
    stage("update service by built image"){
        sh "docker service update --image ${IMAGE_FULL_NAME} --update-parallelism 2  --update-delay 2s ${SERVICE_NAME}"
    }



//     stage("tag image"){
//         sh "docker image tag ${IMAGE_NAME_WITH_INITIAL_TAG} ${IMAGE_FULL_NAME}"
//     }



    // stage("clean previous image and container. Deprecated: 该功能不需要了, 因为现在是Docker Service "){
    //     sh "docker container rm -f ${CONTAINER_NAME}"
//         sh "docker image rm ${IMAGE_NAME_WITH_TAG}"
//         sh "docker image rm ${IMAGE_TO_RUN}"
    // }
//     stage( "pull image" ){
//         sh "docker pull  lyklove/${IMAGE_NAME_WITH_TAG}"
//     }
//    stage("run container volatile_frontend") {
//        sh "docker image ls"
//        sh "docker container run --name ${CONTAINER_NAME} --net=host -d  ${IMAGE_TO_RUN}"
//    }

    // stage("update service by built image"){
    //     sh "docker service update --image ${IMAGE_TO_RUN} --update-parallelism 2  --update-delay 2s ${SERVICE_NAME}"
    // }

    // //Gitlab
    // stage("signal github: deployed"){
    //     echo 'Notify GitLab'
    //     updateGitlabCommitStatus name: 'build', state: 'pending'
    //     updateGitlabCommitStatus name: 'build', state: 'success'
    // }
}

```

## Backend



```groovy
node("slave1") {
    def workspace = pwd()

    def git_branch = 'master'
//    def git_repository = 'git@git.nju.edu.cn:191820133/backend-volatile.git' //Gitlab
    def git_repository = 'git@github.com:VolatileReborn/Backend-VolatileReborn.git' //Github

    def vm_ip = '124.222.135.47'
    def vm_port = '22'
    def vm_user = 'lyk'

    def vm_project_place = "/usr/local/src"
    def vm_target_place = "/usr/local/src/target/"


    def __PROJECT_NAME = 'volatile_reborn'

    def __PROJECT_TYPE = 'backend'

    def __DOCKERHUB_ACCOUNT = 'lyklove'
    def __IMAGE_TAG = 'latest-linux'

    // PORT:
    def COLLECT_CONTAINER_PORT = 9000
    def COLLECT_HOST_PORT = 9000

    def EUREKA_CONTAINER_PORT = 8001
    def EUREKA_HOST_PORT = 8001
    //=======================

    def ORIGINAL_COLLECT_IMAGE_NAME = __PROJECT_TYPE + '_' + __PROJECT_NAME //backend_volatile_reborn

    def IMAGE_NAME_WITH_INITIAL_TAG = ORIGINAL_COLLECT_IMAGE_NAME + ':' + __IMAGE_TAG //backend_volatile_reborn:latest-linux
    def COLLECT_IMAGE_FULL_NAME = __DOCKERHUB_ACCOUNT + '/' + IMAGE_NAME_WITH_INITIAL_TAG // lyklove/backend_volatile_reborn:latest-linux

    def EUREKA_IMAGE_FULL_NAME = "lyklove/backend_eureka_volatile_reborn:latest-linux"


    //IMAGE TO RUN
    def COLLECT_IMAGE_TO_RUN = COLLECT_IMAGE_FULL_NAME
    def EUREKA_IMAGE_TO_RUN = EUREKA_IMAGE_FULL_NAME

    //======================

    //CONTAINER NAME
    def COLLECT_CONTAINER_NAME = ORIGINAL_COLLECT_IMAGE_NAME //backend_volatile_reborn
    def EUREKA_CONTAINER_NAME = "backend_eureka_volatile_reborn"
    //===================

    //SERVICE_NAME
    def COLLECT_SERVICE_NAME = COLLECT_CONTAINER_NAME + '_svc' //backend_volatile_reborn_svc
    def EUREKA_SERVICE_NAME = EUREKA_CONTAINER_NAME + '_svc' //backend_eureka_volatile_reborn_svc
    //============


    stage('clone from github into slave\'s workspace. Using branch: ' + "master") {
        echo "workspace: ${workspace}"
        git branch: "${git_branch}", url: "${git_repository}"
    }


    stage('cd to build context') {
        echo "the context now is:"
        sh "ls -al"
        sh "cd ${workspace}"
        echo "cd to build context, now the context is:"
        sh "ls -al"

    }

    //需要服务器安装maven
    stage('build jar on slave machine, inorder to generate jacoco file ') {

        sh 'mvn --version'

        sh './build.sh'
//        sh "mvn  clean package -X org.jacoco:jacoco-maven-plugin:report  -Dmaven.test.failure.ignore=true"
//        sh "mvn  clean package"

//        echo "build finish on ${vm_ip}"

    }

    // temporarily CLOSE
//    stage( 'testing, using jacoco' ) {
//        jacoco (
//                execPattern: '**/target/jacoco.exec',
//                classPattern: '**/classes',
//                sourcePattern: '**/src/main/java',
//                exclusionPattern: '**/src/test*',
////                inclusionPattern: '**/com/hel/auto/service/*.class,**/com/hel/auto/controller/*.class',
//        )
//    }


    stage("build docker image, tag it"){

        def EUREKA_SERVICE_DOCKERFILE_PATH = './eureka-server/Dockerfile'
        sh "docker build -t ${EUREKA_IMAGE_FULL_NAME} -f ${EUREKA_SERVICE_DOCKERFILE_PATH} ."
        echo "Eureka build finished"


        def COLLECT_SERVICE_DOCKERFILE_PATH = './collect-service/Dockerfile'
        sh "docker build -t ${COLLECT_IMAGE_FULL_NAME} -f ${COLLECT_SERVICE_DOCKERFILE_PATH} ."
        echo "Collect build finished"


    }

    stage("login to dockerhub"){
        withCredentials([usernamePassword(credentialsId: 'DOCKERHUB_KEY', passwordVariable: 'password', usernameVariable: 'username')]) {
            sh 'docker login -u $username -p $password'
        }
    }

    stage("push to dockerhub"){
        echo "begin push to dockerhub"
        sh "docker image push ${COLLECT_IMAGE_FULL_NAME}"
        sh "docker image push ${EUREKA_IMAGE_FULL_NAME}"

    }




        stage("clean previous image and container"){
//        sh "docker container rm -f ${EUREKA_CONTAINER_NAME}"
        sh "docker container rm -f ${COLLECT_CONTAINER_NAME}"

//        sh "docker image rm ${EUREKA_IMAGE_TO_RUN}"
//        sh "docker image rm ${COLLECT_IMAGE_TO_RUN}"
    }


    //Using docker service on Eureka
    //需要先在服务器上手动创建该service
    stage("update Eureka service by built image"){

        sh "docker service update --image ${EUREKA_IMAGE_TO_RUN} --update-parallelism 1  --update-delay 2s ${EUREKA_SERVICE_NAME}"
    }

    stage("run Collect container") {
//        sh "docker image ls"

//        sh "docker container run -p ${EUREKA_HOST_PORT}:${EUREKA_CONTAINER_PORT} --name ${EUREKA_CONTAINER_NAME}   -d ${EUREKA_IMAGE_TO_RUN}"
        sh "docker container run --net=host --restart unless-stopped --name ${COLLECT_CONTAINER_NAME}   -d ${COLLECT_IMAGE_TO_RUN}"

    }




//    //Using docker service
//    //需要先在服务器上手动创建该service
//    stage("update service by built image"){
//
//        sh "docker service update --image ${EUREKA_IMAGE_TO_RUN} --update-parallelism 1  --update-delay 2s ${EUREKA_SERVICE_NAME}"
//        sh "docker service update --image ${COLLECT_IMAGE_TO_RUN} --update-parallelism 1  --update-delay 2s ${COLLECT_SERVICE_NAME}"
//    }




//    stage( "pull image" ){
//        sh "docker pull  lyklove/${IMAGE_NAME_WITH_TAG}"
//    }

//    stage("signal gitlab: deployed"){
//        updateGitlabCommitStatus name: 'deployed', state: 'success'
//    }


}
```



# Agent

* Jenkins通过SSH来连接其他主机, 使其作为agent来进行构建. 本项目就使用宿主机作为slave

* 节点必须已安装了构建所需的环境，比如对应版本的maven, jdk, node, npm, vue等. 整个脚本都在该节点执行

  ```groovy
  node("slave1") {
      //....
  }
  ```



 