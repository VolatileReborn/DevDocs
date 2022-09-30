# 数据库配置说明

* 项目使用mysql数据库, 运行在lyk的server:

  * host: 123.56.20.222

* 用户密码以及权限我已经配置好了:

  * username: VR
  * password: VR
  * 权限: 拥有全部权限

* 目前数据库里什么都没有, 需要后端根据需求自己建数据库和表. 当然, 我之前的项目已经提供了现成的建表语句, 只需要跑一下就行了:

  ![image-20220930230102774](./assets/建表语句.png)

* 注意, 这次我们的项目是“volatile_reborn”, 因此数据库全都用“VR”作为开头名, 比如, 用户数据库名字应该叫叫"VR_user", 而不是"volatile_user", 你们需要自己修改建表语句. 写的时候要小心.

* 可以使用专用工具Datagrip来操作数据库

* 我已经测试了数据库连接, 是OK的:

  ![image-20220930230242925](./assets/数据库连接成功.png)

# Frontend

* 仅存的问题是: Jenkins服务器太垃圾了, 每次cicd都会在npm build那里卡上两小时. 由于jenkins和前端项目都部署在jenkins的宿主机, 所以宿主机卡死会导致jenkins和前端都不能访问....
  * 所以千万不要随意push到master, 可以push到别的分支. 等到发布的那个版本再push到master