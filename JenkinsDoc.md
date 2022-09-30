# Basic

Jenkins地址： http://124.222.135.47:8081/job/backend-volatile/

用户名： lyk

密码： 123456



 我们的Jenkins有三个Item, 前后端和算法服务各一个。因此有三份Jenkinsfile.



# agent

我们通过ssh连接了另一台服务器作为Jenkins节点，指挥该节点进行构建。 该节点已经安装了构建所需的环境，比如对应版本的maven, jdk, node, npm, vue等。 整个脚本都在该节点执行

# docker

我们的项目全部打包成docker容器，push到镜像仓库进行管理。 要运行服务，只需要pull镜像并且启动

# 执行逻辑

前后端的执行逻辑都相同:

1. 从git仓库clone代码
2. cd进入工作目录
3. 预处理， 比如前端需要npm install下载依赖
4. 进行项目构建：

- 后端： maven clean     package
- 前端： npm run build

1. 将构建后的文件打成docker镜像
2. 登录Dockerhub
3. 将镜像push到Dockerhub
4. 从Dockerhub pull镜像
5. 运行镜像

算法部分的执行逻辑略有不同， 因为Python项目要依赖各种库，打包后镜像太大了（有4.5G）， 每次CICD时，这么大的镜像要push到Dockerhub，再pull下来，是巨大的性能瓶颈。 因此算法部分的pipeline脚本，没有了push和pull镜像的操作

同时，对于Python服务，需要专门指定一个`requirement.txt`文件来描述依赖，该文件会被pip3读取（ 这一步在Dockerfile中 ），来下载依赖

**后端脚本**

node("slave1") {

  def workspace = pwd()

 

  def git_branch = 'master'

  def git_repository = 'git@git.nju.edu.cn:191820133/backend-volatile.git'

  def vm_ip = '124.222.135.47'

  def vm_port = '22'

  def vm_user = 'lyk'

 

  def vm_project_place = "/usr/local/src"

  def vm_target_place = "/usr/local/src/target/"

 

 

  def IMAGE_NAME = 'volatile_backend'

  def IMAGE_NAME_WITH_TAG = 'volatile_backend:latest'

  def IMAGE_TO_RUN = 'lyklove/volatile_backend:latest'

  def CONTAINER_NAME = 'volatile_backend'

 

  stage('clone from gitlab into slave\'s workspace') {

​    echo "workspace: ${workspace}"

​    git branch: "${git_branch}", url: "${git_repository}"

  }

 

 

  stage('cd to build context') {

​    echo "the context now is:"

​    sh "ls -al"

​    sh "cd ${workspace}"

​    echo "cd to build context, now the context is:"

​    sh "ls -al"

 

  }

  stage('build jar on slave machine, jacoco file generated') {

 

​    sh 'mvn --version'

​    sh "mvn clean package jacoco:report -Dmaven.test.failure.ignore=true"

​    echo "build finish on ${vm_ip}"

 

  }

 

  stage( 'testing, using jacoco' ) {

​    jacoco (

​        execPattern: '**/target/jacoco.exec',

​        classPattern: '**/classes',

​        sourcePattern: '**/src/main/java',

​        exclusionPattern: '**/src/test*',

//        inclusionPattern: '**/com/hel/auto/service/*.class,**/com/hel/auto/controller/*.class',

​    )

  }

 

 

  stage("build docker image"){

​    sh "docker build -t ${IMAGE_NAME} ."

  }

 

  stage("login to dockerhub"){

​    withCredentials([usernamePassword(credentialsId: 'DOCKERHUB_KEY', passwordVariable: 'password', usernameVariable: 'username')]) {

​      sh 'docker login -u $username -p $password'

​    }

  }

 

  stage("push to dockerhub"){

​    echo "begin push to dockerhub"

​    sh "docker image tag ${IMAGE_NAME_WITH_TAG} lyklove/${IMAGE_NAME_WITH_TAG}"

​    sh "docker image push lyklove/${IMAGE_NAME_WITH_TAG}"

  }

  stage("clean previous image and container"){

​    sh "docker container rm -f ${CONTAINER_NAME}"

​    sh "docker image rm ${IMAGE_NAME_WITH_TAG}"

​    sh "docker image rm ${IMAGE_TO_RUN}"

  }

  stage( "pull image" ){

​    sh "docker pull lyklove/${IMAGE_NAME_WITH_TAG}"

  }

  stage("run container") {

​    sh "docker image ls"

​    sh "docker container run --name ${CONTAINER_NAME} --net=host -d ${IMAGE_TO_RUN}"

  }

  stage("signal gitlab: deployed"){

​    updateGitlabCommitStatus name: 'deployed', state: 'success'

  }

 

 

}

**前端脚本**

node("slave1") {

  def workspace = pwd()

 

  def git_branch = 'master'

  def git_repository = 'git@git.nju.edu.cn:191820133/frontend-volatile.git'

  def vm_ip = '124.222.135.47'

  def vm_port = '22'

  def vm_user = 'lyk'

 

  def vm_project_place = "/usr/local/src"

  def vm_target_place = "/usr/local/src/target/"

 

 

  def IMAGE_NAME = 'volatile_frontend'

  def IMAGE_NAME_WITH_TAG = 'volatile_frontend:latest'

  def IMAGE_TO_RUN = 'lyklove/volatile_frontend:latest'

  def CONTAINER_NAME = 'volatile_frontend'

 

  stage('clone from gitlab into slave\'s workspace') {

​    echo "workspace: ${workspace}"

​    git branch: "${git_branch}", url: "${git_repository}"

  }

 

 

  stage('cd to build context') {

​    echo "the context now is:"

​    sh "ls -al"

​    sh "cd ${workspace}"

​    echo "cd to build context, now the context is:"

​    sh "ls -al"

 

  }

  stage('get version info'){

​    sh 'node -v'

​    sh 'npm -v'

​    sh 'vue -V'

​    sh 'npm list vue'

  }

  stage('build with npm') {

 

//     sh 'npm config set registry http://registry.cnpmjs.org'

​    sh 'npm install'

​    sh 'npm run build'

​    echo "build finish on ${vm_ip}"

  }

 

//   stage('npm run serve'){

//

//     echo 'not using docker yet!!'

//     sh 'npm run serve'

//   }

 

 

  stage("build docker image"){

​    sh "docker build -t ${IMAGE_NAME} --no-cache ."

//     sh "imageId=`docker images | grep #{IMAGE_NAME} | awk '{print $3}'`"

  }

 

  stage("login to dockerhub"){

​    withCredentials([usernamePassword(credentialsId: 'DOCKERHUB_KEY', passwordVariable: 'password', usernameVariable: 'username')]) {

​      sh 'docker login -u $username -p $password'

​    }

  }

 

  stage("push to dockerhub"){

​    echo "begin push to dockerhub"

​    sh "docker image tag ${IMAGE_NAME_WITH_TAG} lyklove/${IMAGE_NAME_WITH_TAG}"

​    sh "docker image push lyklove/${IMAGE_NAME_WITH_TAG}"

  }

  stage("clean previous image and container"){

​    sh "docker container rm -f ${CONTAINER_NAME}"

​    sh "docker image rm ${IMAGE_NAME_WITH_TAG}"

​    sh "docker image rm ${IMAGE_TO_RUN}"

  }

  stage( "pull image" ){

​    sh "docker pull lyklove/${IMAGE_NAME_WITH_TAG}"

  }

  stage("run container") {

​    sh "docker image ls"

​    sh "docker container run --name ${CONTAINER_NAME} --net=host -d ${IMAGE_TO_RUN}"

  }

  stage("signal gitlab: deployed"){

​    updateGitlabCommitStatus name: 'deployed', state: 'success'

  }

 

 

}

 

**算法脚本**

node("slave1") {

  def workspace = pwd()

 

  def git_branch = 'master'

  def git_repository = 'git@git.nju.edu.cn:191820133/ai-volatile.git'

  def vm_ip = '124.222.135.47'

  def vm_port = '22'

  def vm_user = 'lyk'

 

  def vm_project_place = "/usr/local/src"

  def vm_target_place = "/usr/local/src/target/"

 

 

  def IMAGE_NAME = 'volatile_ai'

  def IMAGE_NAME_WITH_TAG = 'volatile_ai:latest'

  def IMAGE_TO_RUN = 'lyklove/volatile_ai:latest'

  def CONTAINER_NAME = 'volatile_ai'

 

  stage('clone from gitlab into slave\'s workspace') {

​    echo "workspace: ${workspace}"

​    git branch: "${git_branch}", url: "${git_repository}"

  }

 

 

  stage('cd to build context') {

​    echo "the context now is:"

​    sh "ls -al"

​    sh "cd ${workspace}"

​    echo "cd to build context, now the context is:"

​    sh "ls -al"

 

  }

 

 

 

  stage("build docker image"){

​    sh "docker build -t ${IMAGE_NAME} ."

  }

 

//   stage("login to dockerhub"){

//     withCredentials([usernamePassword(credentialsId: 'DOCKERHUB_KEY', passwordVariable: 'password', usernameVariable: 'username')]) {

//       sh 'docker login -u $username -p $password'

//     }

//   }

//

  stage("push to dockerhub"){

//     echo "begin push to dockerhub"

​    sh "docker image tag ${IMAGE_NAME_WITH_TAG} lyklove/${IMAGE_NAME_WITH_TAG}"

//     sh "docker image push lyklove/${IMAGE_NAME_WITH_TAG}"

  }

  stage("clean previous image and container"){

​    sh "docker container rm -f ${CONTAINER_NAME}"

//     sh "docker image rm ${IMAGE_NAME_WITH_TAG}"

//     sh "docker image rm ${IMAGE_TO_RUN}"

  }

//   stage( "pull image" ){

//     sh "docker pull lyklove/${IMAGE_NAME_WITH_TAG}"

//   }

  stage("run container") {

​    sh "docker image ls"

​    sh "docker container run --name ${CONTAINER_NAME} --net=host -d ${IMAGE_TO_RUN}"

  }

  stage("signal gitlab: deployed"){

​    updateGitlabCommitStatus name: 'deployed', state: 'success'

  }

 

 

}

 

 