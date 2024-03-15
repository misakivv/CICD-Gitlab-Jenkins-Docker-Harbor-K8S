[toc]

> 本篇文章参考 [山河已无恙大佬的文章：（持续集成部署Hexo博客Demo）](https://blog.csdn.net/sanhewuyang/article/details/121189389)

## 涉及内容:

- `Gitlab`+`Jenkins`+`Docker`+`Harbor`+`K8S集群` 的`CICD`搭建教程
- 在搭建好的`CICD`平台上`持续集成部署hexo博客系统`

- 其中`Gitlab`+`Jenkins` +`Harbor`都是通过`容器化`部署
- 篇幅有限，关于CD环境`k8s集群`这里用之前部署好的，并且已经做了`kubeconfig`证书
- 下面为涉及到的机器：

|      用到的机器       |       ip       |
| :-------------------: | :------------: |
|        客户机         |   本地物理机   |
| Gitlab+Jenkins+Docker | 192.168.112.10 |
| docker镜像仓库:harbor | 192.168.112.20 |
|  k8s集群-master节点   | 192.168.112.30 |
|   k8s集群-node节点    | 192.168.112.40 |
|   k8s集群-node节点    | 192.168.112.50 |

|                            拓扑图                            |
| :----------------------------------------------------------: |
| 这里`客户机`用本地的`IDE持续编码`，然后`push`代码到`gitlab`，`gitlab`中的`web钩子`触发`jenkins`中配置好的`构建触发器`，通过`shell命令`拉取`gitlab仓库中的代码`，然后通过拉取的`应用源码`和`Dockerfile`文件来构建`应用镜像`，构建完成后将`应用镜像push到harbor私有镜像仓库`，然后通过`shell`命令的方式在`jenkins`中用`kubelet客户端`将`镜像`从私有仓库拉取到`k8s集群`并更新其`deploy`中的镜像,默认`deploy`更新副本的方式为`滚动更新`，整个流程中，只有客户机push代码是手手动的方式，其他全是自动 |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133224209-1645744318.png) |

------

## 一、CICD服务器环境搭建

**CI即为`持续集成(Continue Integration,简称CI)`，用通俗的话讲，就是`持续的整合版本库代码编译后制作应用镜像`。建立有效的持续集成环境可以减少开发过程中一些不必要的问题、`提高代码质量、快速迭代`等,**

> 常用的工具和平台有:

`Jenkins`:基于Java开发的一种[持续集成](https://so.csdn.net/so/search?q=持续集成&spm=1001.2101.3001.7020)工具,用于监控持续重复的工作,旨在提供一个开放易用的软件平台,使软件的持续集成变成可能。
`Bamboo`: 是一个企业级商用软件,可以部署在大规模生产环境中。

**CD即持续交付Continuous Delivery和持续部署Continuous Deployment，用通俗的话说，即可以持续的部署到生产环境给客户使用，这里分为两个阶段，持续交付我理解为满足上线条件的过程，但是没有上线，持续部署，即为上线应用的过程**

关于`CD环境`，我们使用以前搭建好的`K8s集群`，[K8s](https://so.csdn.net/so/search?q=K8s&spm=1001.2101.3001.7020)集群可以实现应用的`健康检测，动态扩容，滚动更新`等优点，关于K8s集群的搭建，小伙伴可以看看我的其他文章

> 我们来搭建CI服务器:操作服务器： jenkins：192.168.112.10

### 1、docker 环境安装

#### （1）、拉取镜像，启动并设置开机自启

```bash
[root@jenkins ~]# systemctl start docker
[root@jenkins ~]# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
```

![image-20240308222458972](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133223879-593493274.png)

#### （2）、配置docker加速器

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://2tefyfv7.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

![image-20240308223232016](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133223516-1007407259.png)

### 2、安装并配置GitLab

GitLab是一个基于Git的版本控制平台，,提供了Git仓库管理、代码审查、问题跟踪、活动反馈和wiki，当然同时也提供了

```bash
[root@jenkins ~]# docker pull beginor/gitlab-ce
```

![image-20240308223537309](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133223201-709349606.png)

#### （1）、创建共享卷目录

```bash
[root@jenkins ~]# mkdir -p /data/gitlab/etc/ /data/gitlab/log/ /data/gitlab/data
[root@jenkins ~]# chmod 777 /data/gitlab/etc/ /data/gitlab/log/ /data/gitlab/data/
```

#### （2）、创建 gitlab 容器

```bash
[root@jenkins ~]# docker run -itd --name=gitlab --restart=always --privileged=true   -p 8443:443  -p 80:80 -p 222:22 -v  /data/gitlab/etc:/etc/gitlab -v  /data/gitlab/log:/var/log/gitlab -v  /data/gitlab/data:/var/opt/gitlab  beginor/gitlab-ce
805eb9eac8367c53a8d458fec17649e3b3b206f3dc74c99c7a037a41dd9e8ca6
[root@jenkins ~]# docker ps
CONTAINER ID   IMAGE               COMMAND             CREATED          STATUS                             PORTS                                                                                                             NAMES
805eb9eac836   beginor/gitlab-ce   "/assets/wrapper"   20 seconds ago   Up 19 seconds (health: starting)   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:222->22/tcp, :::222->22/tcp, 0.0.0.0:8443->443/tcp, :::8443->443/tcp   gitlab
```

> **切记:这里的端口要设置成80，要不push项目会提示没有报错，如果宿主机端口被占用，需要把这个端口腾出来**

#### (3)、关闭容器修改配置文件

```bash
[root@jenkins ~]# docker stop gitlab
gitlab
```

**external_url 'http://192.168.112.10'**

```bash
[root@jenkins ~]# cat /data/gitlab/etc/gitlab.rb |grep external_url
##! For more details on configuring external_url see:
# external_url 'GENERATED_EXTERNAL_URL'
# registry_external_url 'https://registry.gitlab.example.com'
# pages_external_url "http://pages.example.com/"
# gitlab_pages['artifacts_server_url'] = nil # Defaults to external_url + '/api/v4'
# mattermost_external_url 'http://mattermost.example.com'
[root@jenkins ~]# sed -i "/external_url 'GENERATED_EXTERNAL_URL'/a external_url\t'http://192.168.112.10' "  /data/gitlab/etc/gitlab.rb
[root@jenkins ~]# cat /data/gitlab/etc/gitlab.rb |grep external_url
##! For more details on configuring external_url see:
# external_url 'GENERATED_EXTERNAL_URL'
external_url    'http://192.168.112.10'
# registry_external_url 'https://registry.gitlab.example.com'
# pages_external_url "http://pages.example.com/"
# gitlab_pages['artifacts_server_url'] = nil # Defaults to external_url + '/api/v4'
# mattermost_external_url 'http://mattermost.example.com'

```

**gitlab_rails[‘gitlab_ssh_host’] = '192.168.112.10'**

```bash
[root@jenkins ~]# cat /data/gitlab/etc/gitlab.rb |grep gitlab_ssh_host
# gitlab_rails['gitlab_ssh_host'] = 'ssh.host_example.com'
[root@jenkins ~]# sed -i "/gitlab_ssh_host/a gitlab_rails['gitlab_ssh_host'] = '192.168.112.10' "  /data/gitlab/etc/gitlab.rb
[root@jenkins ~]# cat /data/gitlab/etc/gitlab.rb |grep gitlab_ssh_host                   # gitlab_rails['gitlab_ssh_host'] = 'ssh.host_example.com'
gitlab_rails['gitlab_ssh_host'] = '192.168.112.10'
```

**gitlab_rails[gitlab_shell_ssh_port] = 222**

```bash
[root@jenkins ~]# cat /data/gitlab/etc/gitlab.rb | grep gitlab_shell_ssh
# gitlab_rails['gitlab_shell_ssh_port'] = 22
[root@jenkins ~]# sed -i "/gitlab_shell_ssh_port/a gitlab_rails['gitlab_shell_ssh_port'] = 222" /data/gitlab/etc/gitlab.rb
[root@jenkins ~]# cat /data/gitlab/etc/gitlab.rb | grep gitlab_shell_ssh                 # gitlab_rails['gitlab_shell_ssh_port'] = 22
gitlab_rails['gitlab_shell_ssh_port'] = 222
[root@jenkins ~]# vim /data/gitlab/data/gitlab-rails/etc/gitlab.yml
  ## GitLab settings
  gitlab:
    ## Web server settings (note: host is the FQDN, do not include http://)
    host: 192.168.112.10
    port: 80
    https: false
```

#### （4）、修改完配置文件之后。直接启动容器

```bash
[root@jenkins ~]# docker start gitlab
gitlab
[root@jenkins ~]# docker ps
CONTAINER ID   IMAGE               COMMAND             CREATED          STATUS                            PORTS                                                                                                             NAMES
805eb9eac836   beginor/gitlab-ce   "/assets/wrapper"   21 minutes ago   Up 7 seconds (health: starting)   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:222->22/tcp, :::222->22/tcp, 0.0.0.0:8443->443/tcp, :::8443->443/tcp   gitlab
```

|                            Gitlab                            |
| :----------------------------------------------------------: |
| **在宿主机所在的物理机访问，`http://192.168.112.10/` ，会自动跳转到修改密码(root用户),如果密码设置的没有满足一定的复杂性，则会报500，需要从新设置** |
| ![image-20240309095514791](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133222823-987962708.png) |
| ![image-20240309095626951](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133222431-54215356.png) |
|                      **登录进入仪表盘**                      |
| ![image-20240309095724952](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133222073-98454301.png) |
| ![image-20240309095833483](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133221721-1921736997.png) |
| ![image-20240309095940387](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133221347-62003761.png) |
| **然后我们简单测试一下，push一个项目上去,会提示输入用户密码,这里的项目是一个基于hexo的博客系统** |
| ![image-20240309101725278](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133220965-324171227.png) |
|                    **项目成功上传Gitlab**                    |
| ![image-20240309112617557](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133220629-747513728.png) |
| ![image-20240309112800078](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133220269-999570774.png) |

#### （5）、相关的git命令（针对已存在的文件夹）

```bash
cd existing_folder
git init
git remote add origin http://192.168.112.10/root/hexo-gitlab-blog.git
git add .
git commit -m "Initial commit"
git push -u origin master
```

### 3、安装配置远程镜像仓库harbor

下面我们要配置私有的docker镜像仓库，用到的机器为：

> 操作服务器: harbor:192.168.112.20

这里仓库我们选择`harbor`，因为有web页面，当然也可以使用 `registry`

**harbor的配置**

| harbor的安装使用步骤                                |
| --------------------------------------------------- |
| 安装并启动docker并安装docker-compose                |
| 上传harbor的离线包                                  |
| 导入harbor的镜像                                    |
| 编辑harbor.yml                                      |
| 修改hostname 为自己的主机名,不用证书需要注释掉https |
| harbor_admin_password 登录密码                      |
| 安装compose                                         |
| 运行脚本 ./install.sh                               |
| 在浏览器里输入IP访问                                |
| docker login IP --家目录下会有一个.docker文件夹     |

> 下面我们开始安装

#### （1）、首先需要设置selinux、防火墙

```bash
[root@harbor ~]# getenforce
Disabled
[root@harbor ~]# systemctl disable firewalld.service --now
```

#### （2）、安装并启动docker并安装docker-compose，关于docker-compose，这里不用了解太多，一个轻量的docker编排工具

```bash
yum install -y docker-ce
yum install -y docker-compose
```

#### （3）、解压harbor 安装包：harbor-offline-installer-v2.0.6.tgz，导入相关镜像

**harbor安装包：**[harbor](https://github.com/goharbor/harbor/tags)

```bash
[root@harbor ~]# ls
aliyun.sh  anaconda-ks.cfg  harbor-offline-installer-v2.0.6.tgz
[root@harbor ~]# tar -zxvf harbor-offline-installer-v2.0.6.tgz
harbor/harbor.v2.0.6.tar.gz
harbor/prepare
harbor/LICENSE
harbor/install.sh
harbor/common.sh
harbor/harbor.yml.tmpl
[root@harbor ~]# docker load -i harbor/harbor.v2.0.6.tar.gz
```

#### （4）、修改配置文件

```bash
[root@harbor ~]# cd harbor/
[root@harbor harbor]# ls
common.sh  harbor.v2.0.6.tar.gz  harbor.yml.tmpl  install.sh  LICENSE  prepare
[root@harbor harbor]# cp harbor.yml.tmpl harbor.yml
[root@harbor harbor]# ls
common.sh             harbor.yml       install.sh  prepare
harbor.v2.0.6.tar.gz  harbor.yml.tmpl  LICENSE
[root@harbor harbor]# vim harbor.yml
```

#### （5）、harbor.yml：设置IP和用户名密码

```bash
# Configuration file of Harbor

# The IP address or hostname to access admin UI and registry service.
# DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname: 192.168.112.20

# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80

# https related config
#https:
  # https port for harbor, default is 443
#  port: 443
  # The path of cert and key files for nginx
#  certificate: /your/certificate/path
#  private_key: /your/private/key/path

# # Uncomment following will enable tls communication between all harbor components
# internal_tls:
#   # set enabled to true means internal tls is enabled
#   enabled: true
#   # put your cert and key files on dir
#   dir: /etc/harbor/tls/internal

# Uncomment external_url if you want to enable external proxy
# And when it enabled the hostname will no longer used
# external_url: https://reg.mydomain.com:8433

# The initial password of Harbor admin
# It only works in first time to install harbor
# Remember Change the admin password from UI after launching Harbor.
harbor_admin_password: Harbor12345
```

#### （6）、`./prepare && ./install.sh`

```bash
[root@harbor harbor]# ./prepare
prepare base dir is set to /root/harbor
WARNING:root:WARNING: HTTP protocol is insecure. Harbor will deprecate http protocol in the future. Please make sure to upgrade to https
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/registryctl/config.yml
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
Generated and saved secret to file: /data/secret/keys/secretkey
Successfully called func: create_root_cert
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir
```

```bash
[root@harbor harbor]# ./install.sh

[Step 0]: checking if docker is installed ...

Note: docker version: 25.0.4

[Step 1]: checking docker-compose is installed ...

Note: docker-compose version: 1.18.0

[Step 2]: loading Harbor images ...
Loaded image: goharbor/notary-server-photon:v2.0.6
Loaded image: goharbor/clair-photon:v2.0.6
Loaded image: goharbor/clair-adapter-photon:v2.0.6
Loaded image: goharbor/harbor-portal:v2.0.6
Loaded image: goharbor/harbor-core:v2.0.6
Loaded image: goharbor/harbor-db:v2.0.6
Loaded image: goharbor/harbor-jobservice:v2.0.6
Loaded image: goharbor/redis-photon:v2.0.6
Loaded image: goharbor/notary-signer-photon:v2.0.6
Loaded image: goharbor/harbor-log:v2.0.6
Loaded image: goharbor/harbor-registryctl:v2.0.6
Loaded image: goharbor/trivy-adapter-photon:v2.0.6
Loaded image: goharbor/chartmuseum-photon:v2.0.6
Loaded image: goharbor/prepare:v2.0.6
Loaded image: goharbor/nginx-photon:v2.0.6
Loaded image: goharbor/registry-photon:v2.0.6


[Step 3]: preparing environment ...

[Step 4]: preparing harbor configs ...
prepare base dir is set to /root/harbor
WARNING:root:WARNING: HTTP protocol is insecure. Harbor will deprecate http protocol in the future. Please make sure to upgrade to https
Clearing the configuration file: /config/log/logrotate.conf
Clearing the configuration file: /config/log/rsyslog_docker.conf
Clearing the configuration file: /config/nginx/nginx.conf
Clearing the configuration file: /config/core/env
Clearing the configuration file: /config/core/app.conf
Clearing the configuration file: /config/registry/passwd
Clearing the configuration file: /config/registry/config.yml
Clearing the configuration file: /config/registryctl/env
Clearing the configuration file: /config/registryctl/config.yml
Clearing the configuration file: /config/db/env
Clearing the configuration file: /config/jobservice/env
Clearing the configuration file: /config/jobservice/config.yml
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/registryctl/config.yml
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
Creating harbor-log ... done
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir


Creating registry ... done
Creating harbor-core ... done
Creating network "harbor_harbor" with the default driver
Creating nginx ... done
Creating harbor-db ...
Creating redis ...
Creating registryctl ...
Creating registry ...
Creating harbor-portal ...
Creating harbor-core ...
Creating nginx ...
Creating harbor-jobservice ...
✔ ----Harbor has been installed and started successfully.----
```

#### （7）、查看相关的镜像

```bash
[root@harbor harbor]# docker ps
CONTAINER ID   IMAGE                                COMMAND                   CREATED         STATUS                   PORTS                                   NAMES
9572b7a8d0a8   goharbor/harbor-jobservice:v2.0.6    "/harbor/entrypoint.…"   5 minutes ago   Up 5 minutes (healthy)                                           harbor-jobservice
83b679a70258   goharbor/nginx-photon:v2.0.6         "nginx -g 'daemon of…"   5 minutes ago   Up 5 minutes (healthy)   0.0.0.0:80->8080/tcp, :::80->8080/tcp   nginx
e7c53195c856   goharbor/harbor-core:v2.0.6          "/harbor/entrypoint.…"   5 minutes ago   Up 5 minutes (healthy)                                           harbor-core
37884d3bb185   goharbor/registry-photon:v2.0.6      "/home/harbor/entryp…"   5 minutes ago   Up 5 minutes (healthy)   5000/tcp                                registry
d4de74c6b397   goharbor/harbor-portal:v2.0.6        "nginx -g 'daemon of…"   5 minutes ago   Up 5 minutes (healthy)   8080/tcp                                harbor-portal
3459fba85f4c   goharbor/harbor-db:v2.0.6            "/docker-entrypoint.…"   5 minutes ago   Up 5 minutes (healthy)   5432/tcp                                harbor-db
febab24100f4   goharbor/redis-photon:v2.0.6         "redis-server /etc/r…"   5 minutes ago   Up 5 minutes (healthy)   6379/tcp                                redis
8b6f3d626464   goharbor/harbor-registryctl:v2.0.6   "/home/harbor/start.…"   5 minutes ago   Up 5 minutes (healthy)                                           registryctl
52a51aae1c1b   goharbor/harbor-log:v2.0.6           "/bin/sh -c /usr/loc…"   5 minutes ago   Up 5 minutes (healthy)   127.0.0.1:1514->10514/tcp               harbor-log
```

#### （8）、访问测试

|                            harbor                            |
| :----------------------------------------------------------: |
| ![image-20240309143956535](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133219897-1694938928.png) |
| ![image-20240309144057990](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133219565-848745049.png) |

### 4、CI服务器的docker配置

**这里因为我们要在192.168.112.10(CI服务器)上push镜像到192.168.112.20(私仓)，所有需要修改CI服务器上的Docker配置。添加仓库地址**

> 操作服务器： jenkins：192.168.112.10

#### （1）、修改配置文件

```bash
[root@jenkins ~]# cat /etc/docker/daemon.json
{
  "registry-mirrors": ["https://2tefyfv7.mirror.aliyuncs.com"]
}
[root@jenkins ~]# vim /etc/docker/daemon.json
```

**修改后的配置文件**

```bash
[root@jenkins ~]# cat /etc/docker/daemon.json
{
  "registry-mirrors": ["https://2tefyfv7.mirror.aliyuncs.com"],
  "insecure-registries": ["192.168.112.20"]
}
```

**加载使其生效**

```bash
[root@jenkins ~]# systemctl daemon-reload
[root@jenkins ~]# systemctl restart docker
```

**CI机器简单测试一下**

```bash
[root@jenkins ~]# docker login 192.168.112.20
Username: admin
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@jenkins ~]# docker tag busybox 192.168.112.20/library/busybox
[root@jenkins ~]# docker images
REPOSITORY                       TAG       IMAGE ID       CREATED       SIZE
192.168.112.20/library/busybox   latest    beae173ccac6   2 years ago   1.24MB
busybox                          latest    beae173ccac6   2 years ago   1.24MB
beginor/gitlab-ce                latest    5595d4ff803e   5 years ago   1.5GB
[root@jenkins ~]# docker push 192.168.112.20/library/busybox
Using default tag: latest
The push refers to repository [192.168.112.20/library/busybox]
01fd6df81c8e: Mounted from library/bysybox
latest: digest: sha256:62ffc2ed7554e4c6d360bce40bbcf196573dd27c4ce080641a2c59867e732dee size: 527
```

#### （2）、push一个镜像，可以在私仓的web页面查看

|                            harbor                            |
| :----------------------------------------------------------: |
| ![image-20240309161634356](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133219214-2096159089.png) |

> 到这里。我们配置了镜像仓库

### 5、安装配置jenkins

> 操作服务器：  jenkins：192.168.112.10

#### （1）、镜像jenkins拉取

```bash
[root@jenkins ~]# docker pull jenkins/jenkins:latest
latest: Pulling from jenkins/jenkins
0e29546d541c: Pull complete
11bbb8c402a7: Pull complete
cf91f018150b: Pull complete
a98e88c6f0f0: Pull complete
f67fc70d671a: Pull complete
edbe48067464: Pull complete
fa23ca93dd6b: Pull complete
00159d993c13: Pull complete
f28fb40a17cf: Pull complete
071d309df04b: Pull complete
78599f36e494: Pull complete
896a32d969fb: Pull complete
3f1a51ea9f7f: Pull complete
26e724f0bfad: Pull complete
b377e1ae1384: Pull complete
d3cdbe7e8b9f: Pull complete
f3b40ebc3458: Pull complete
Digest: sha256:c3fa8e7f70d1e873ea6aa87040c557aa53e6707eb1d5ecace7f6884a87588ac8
Status: Downloaded newer image for jenkins/jenkins:latest
docker.io/jenkins/jenkins:latest
```

#### （2）、创建共享卷，修改所属组和用户,和容器里相同

> 这里为什么要改成 1000，是因为容器里是以 jenkins 用户的身份去读写数据，而在容器里jenkins 的 uid 是 1000

```bash
[root@jenkins ~]# mkdir /jenkins
[root@jenkins ~]# chown 1000:1000 /jenkins
# 这里为什么要改成 1000，是因为容器里是以 jenkins 用户的身份去读写数据，而在容器里jenkins 的 uid 是 1000
```

#### （3）、创建创建 jenkins 容器

```bash
[root@jenkins ~]# docker run -dit -p 8080:8080 -p 50000:50000 --name jenkins  --privileged=true --restart=always -v /jenkins:/var/jenkins_home jenkins/jenkins:latest
f250456a77abeb916eb36781eafd8c17e3aad8ec26d5f6e006df4956d234f445
[root@jenkins ~]# docker ps | grep jenkins
f250456a77ab   jenkins/jenkins:latest   "/sbin/tini -- /usr/…"   17 seconds ago   Up 16 seconds                0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 0.0.0.0:50000->50000/tcp, :::50000->50000/tcp                          jenkins
```

|                         访问jenkins                          |
| :----------------------------------------------------------: |
| ![image-20240309163116129](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133218843-1014131572.png) |
|   **因为要修改 jenkins 的配置，所以此时关闭 jenkins 容器**   |

```bash
[root@jenkins ~]# docker stop jenkins
jenkins
```

#### （4）、更换国内清华大学镜像,Jenkins下载插件特别慢，更换国内的清华源的镜像地址会快不少

```bash
[root@jenkins jenkins]# cat /jenkins/hudson.model.UpdateCenter.xml
<?xml version='1.1' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json</url>
  </site>
</sites>[root@jenkins jenkins]# sed -i  's#updates.jenkins.io/update-center.json#mirrors.nghua.edu.cn/jenkins/updates/update-center.json#g '  /jenkins/hudson.model.UpdateCenter.xml
[root@jenkins jenkins]# cat /jenkins/hudson.model.UpdateCenter.xml                       <?xml version='1.1' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json</url>
  </site>
</sites>
```

**"http://www.google.com/" 替换为 "http://www.baidu.com/"**

```bash
[root@jenkins jenkins]# yum install -y jq
[root@jenkins jenkins]# cat /jenkins/updates/default.json | jq '.connectionCheckUrl'
"https://www.google.com/"
[root@jenkins jenkins]# cat /jenkins/updates/default.json | jq 'keys'
[
  "connectionCheckUrl",
  "core",
  "deprecations",
  "generationTimestamp",
  "id",
  "plugins",
  "signature",
  "updateCenterVersion",
  "warnings"
]
[root@jenkins jenkins]# sed -i    s#http://www.google.com/#http://www.baidu.com/#g  /jenkins/updates/default.json
```

**替换后查看**

```bash
 [root@jenkins jenkins]# cat /jenkins/updates/default.json | jq '.connectionCheckUrl'
"https://www.baidu.com/"
[root@jenkins jenkins]# cat /jenkins/updates/default.json | jq 'keys'                    [
  "connectionCheckUrl",
  "core",
  "deprecations",
  "generationTimestamp",
  "id",
  "plugins",
  "signature",
  "updateCenterVersion",
  "warnings"
]
```

#### （5）、重启docker，获取登录密匙

```bash
[root@jenkins jenkins]# docker start jenkins
jenkins
[root@jenkins jenkins]# cat /jenkins/secret
secret.key                secret.key.not-so-secret  secrets/
[root@jenkins jenkins]# cat /jenkins/secrets/initialAdminPassword
f54e4a2c7dd249ce9f7d4f15121005d8
```

**需要修改jenkins绑定的docker的启动参数**，`ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2376 -H fd:// --containerd=/run/containerd/containerd.sock`

```bash
vim /lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2376 -H fd:// --containerd=/run/containerd/containerd.sock
```

**修改镜像库启动参数后需要重启docker**

```bash
[root@jenkins jenkins]# systemctl daemon-reload
[root@jenkins jenkins]# systemctl restart docker
```

#### （6）、安装 docker 插件

| jenkins相关配置，这里的配置照着图片就好，需要配置一个docker集群供jenkins来根据Dockerfile构建镜像并push到私仓，这里docker集群即为CI服务器的docker |
| ------------------------------------------------------------ |
| ![image-20240309165258162](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133218411-452739561.png) |
| ![image-20240309205437128](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133218028-1180296570.png) |
| ![image-20240309205740569](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133217649-1120493856.png) |
| ![image-20240309170710825](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133217309-485385350.png) |
| ![image-20240309170733207](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133216945-1078604726.png) |
| 依此点击`Manage Jenkins`->`Manage Plugins`->`AVAILABLE`->`Search` 搜索`docker`、`docker-build-step` |
| ![image-20240309205906294](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133216499-1769152759.png) |
| ![image-20240309210102409](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133216144-1550253267.png) |
| ![image-20240309210302041](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133215714-855063050.png) |
| ![image-20240309210737434](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133215327-352128821.png) |
| ![image-20240309210648867](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133214977-1628845243.png) |
| 修改镜像库启动参数，`ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2376 -H fd:// --containerd=/run/containerd/containerd.sock` |
| ![image-20240309211406363](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133214623-112188310.png) |
| ![image-20240309211642888](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133214257-1344416477.png) |
| **关联docker和jenkins**                                      |
| ![image-20240309211941681](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133213887-1398952175.png) |

#### （7）、jenkins 安全设置

**后面 gitlab 要和 jenkins 进行联动，所以必须要需要对 jenkins 的安全做一些设置，依次点击 系统管理-全局安全配置-授权策略，勾选"匿名用户具有可读权限"**

|                                                              |
| :----------------------------------------------------------: |
| ![image-20240309212142618](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133213517-872267659.png) |
| ![image-20240309212308094](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133213138-1206288167.png) |

**添加 JVM 运行参数 `-Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true` 运行跨站请求访问**

```bash
[root@jenkins jenkins]# docker exec -u root -it jenkins /bin/bash
```

#### （8）、下载kubectl客户端工具

> 这里的话我们要通过jenkins上的kubectl客户端连接k8s,所以我们需要安装一个k8s的客户端kubectl，下载k8s客户端

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
yum install kubelet-1.22.2 kubeadm-1.22.2 kubectl-1.22.2 -y
systemctl enable kubelet && systemctl start kubelet
```

##### 拷贝 kubeconfig 文件

**然后拷贝kubeconfig 证书,k8s集群(一主两从)中查看证书位置`/etc/kubernetes/admin.conf`**

```bash
[root@jenkins ~]# scp root@192.168.112.30:/etc/kubernetes/admin.conf .
The authenticity of host '192.168.112.30 (192.168.112.30)' can't be established.
ECDSA key fingerprint is SHA256:d5XrT2DNJojgq53QBNjVvg8JwuYbQyctCh2Bi2l2f0E.
ECDSA key fingerprint is MD5:96:8c:ec:78:63:de:7a:b2:3c:85:8d:5b:9f:f4:94:e8.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.112.30' (ECDSA) to the list of known hosts.
root@192.168.112.30's password:
admin.conf                                             100% 5634     3.6MB/s   00:00
```

##### 拷贝证书和k8s集群客户端工具到jenkins容器内

```bash
[root@jenkins ~]# docker cp admin.conf jenkins:/
Successfully copied 7.68kB to jenkins:/
[root@jenkins ~]# docker cp kubectl jenkins:/
Successfully copied 21.5MB to jenkins:/
```

#### （9）、kubectl命令测试

```bash
./kubectl --kubeconfig=admin.conf get pods -A
```

> 发现没有权限，这里我们为了方便，直接赋予集群中的`cluster-admin`角色

```bash
kubectl create clusterrolebinding  test  --clusterrole=cluster-admin --user=jenkins
```

**命令测试没有问题**

```bash
./kubectl --kubeconfig=admin.conf get pods -A
```

## 二 、hexo博客系统CICD实战

### 1、k8s集群中配置hexo生产环境高可用

**我们要部署`Nginx`来运行`hexo`博客系统，`hexo`编译完后为一堆静态文件，所以我们需要创建一个`svc`和一个`deploy`，使用`SVC`提供服务，使用`deploy`提供服务能力,使用`Nginx+hexo的静态文件`构成的镜像**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginxdep
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: blog
        name: web
        resources:
          requests:
            cpu: 100m
      restartPolicy: Always
```

#### （1）、deployments创建

**这里我们先用一个Nginx镜像来代替hexo博客的镜像**

```bash
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$kubectl apply  -f nginx.yaml
deployment.apps/nginxdep created
```

**查看deployments和pod**

```bash
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$kubectl get deployments.apps  | grep nginxdep
nginxdep                  2/2     2            2           109s
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$kubectl get pods -o wide  | grep web
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$kubectl get pods -o wide  | grep nginxdep
nginxdep-645bf755b9-2w8jv                            1/1     Running   0                 2m22s   10.244.171.164   vms82.liruilongs.github.io   <none>           <none>
nginxdep-645bf755b9-jfqxj                            1/1     Running   0                 2m22s   10.244.171.157   vms82.liruilongs.github.io   <none>           <none>
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$
```

#### （2）、service创建

```bash
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$kubectl expose deploy    nginxdep  --port=8888 --target-port=80 --type=NodePort
service/nginxdep exposed
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$kubectl get svc -o wide | grep nginxdep
nginxdep                            NodePort    10.106.217.50    <none>        8888:31964/TCP                 16s   app=nginx
```

**访问测试没有问题，之后我们配置好jenkins上的触发器，直接替换就OK**

```html
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$curl 127.0.0.1:31964
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$
```

### 2、k8s集群配置私仓地址

**我们通过`kubectl set`命令更新`deploy`的镜像时，获取的镜像是通过私仓获取的，所以需要在启动参数添加私仓地址**

> ExecStart=/usr/bin/dockerd --insecure-registry 192.168.26.56 -H fd:// --containerd=/run/containerd/containerd.sock

**这里所有的节点都需要设置后重启docker**

```bash
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$vim  /usr/lib/systemd/system/docker.service
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$systemctl daemon-reload ;systemctl restart docker &
[1] 23273
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$ssh root@192.168.26.82
Last login: Sun Jan 16 06:09:07 2022 from 192.168.26.1
┌──[root@vms82.liruilongs.github.io]-[~]
└─$vim  /usr/lib/systemd/system/docker.service
┌──[root@vms82.liruilongs.github.io]-[~]
└─$systemctl daemon-reload ;systemctl restart docker &
[1] 26843
┌──[root@vms82.liruilongs.github.io]-[~]
└─$exit
登出
Connection to 192.168.26.82 closed.
```

### 3、jenkins配置CICD流程

**访问jenkins，接下来才是重点，我们要的jenkins上配置整个CICD流程，从而实现自动化**

| **访问jenkins，接下来才是重点，我们要的jenkins上配置整个CICD流程，从而实现自动化** |
| :----------------------------------------------------------: |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133212763-708927107.png) |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133212237-1830614777.png) |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133211891-1118662956.png) |
| 这里的`Token`我们设置为：4bf636c8214b7ff0a0fb，同时需要记住访问方式：`JENKINS_URL/job/liruilong-cicd/build?token=TOKEN_NAME` |
|              构建触发器选择shell构建：克隆代码               |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133211550-1683363634.png) |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133211204-1798545522.png) |
|                         选择镜像构建                         |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133209829-1086192878.png) |
|                      构建镜像并push私仓                      |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133210518-580288931.png) |
| **这里切记需要添加私仓的认证信息，即上面设置的用户名和密码** |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133210180-1915744495.png) |
|                 **选择shell构建，更新镜像**                  |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133209829-1086192878.png) |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133209534-1204916787.png) |

> 相关的文本信息

```bash
cd ~
rm -rf blog
git clone http://192.168.26.55/root/blog.git

/var/jenkins_home/blog/

192.168.26.56/library/blog:${BUILD_NUMBER}

export KUBECONFIG=/kc1;
/kubectl set image deployment/nginxdep  *="192.168.26.56/library/blog:${BUILD_NUMBER}" -n kube-system
```

### 4、配置 gitlab 和 jenkins 的联动

|                      访问gitlab配置联动                      |
| :----------------------------------------------------------: |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133209145-142696047.png) |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133208806-979079301.png) |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133208418-1167257321.png) |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133208041-1495659240.png) |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133207676-1003273510.png) |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133207356-1492659478.png) |
|                       点击增加web钩子                        |
|          /view/all/job/liruilong-cicd/build?token=           |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133207023-1965730277.png) |

> 到这里,联动已经配置完成

### 5、编写Dockerfile文件，更新代码测试

**下面我们编译一下hexo，生成public的一个文件夹,然后上传gitlab**

```bash
  PS F:\blogger> hexo g
  .....
  PS F:\blogger> git add .\public\
  PS F:\blogger> git commit -m "编译代码"
  PS F:\blogger> git push
```

**同时需要编写Dockerfile文件来创建镜像**

```bash
FROM docker.io/library/nginx:latest
MAINTAINER liruilong
ADD ./public/  /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g","daemon off;"]
12345
PS F:\blogger> git add .
PS F:\blogger> git commit -m "Dockcerfile文件编写"
[master 217e0ed] Dockcerfile文件编写
 1 file changed, 1 deletion(-)      
PS F:\blogger> git push 
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 8 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 307 bytes | 307.00 KiB/s, done.
Total 3 (delta 2), reused 0 (delta 0)
To http://192.168.26.55/root/blog.git
   6690612..217e0ed  master -> master
PS F:\blogger> 
```

|                         jenkins输出                          |
| :----------------------------------------------------------: |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133206668-224303473.png) |

```bash
Started by remote host 192.168.26.1
Running as SYSTEM
Building in workspace /var/jenkins_home/workspace/liruilong-cicd
[liruilong-cicd] $ /bin/sh -xe /tmp/jenkins6108687102523328796.sh
+ cd /var/jenkins_home
+ rm -rf blog
+ git clone http://192.168.26.55/root/blog.git
Cloning into 'blog'...
Docker Build
Docker Build: building image at path /var/jenkins_home/blog
Step 1/5 : FROM docker.io/library/nginx:latest


 ---> f8f4ffc8092c

Step 2/5 : MAINTAINER liruilong


 ---> Running in e341b5562b64

Removing intermediate container e341b5562b64

 ---> 4e9f5aa47ab5

Step 3/5 : ADD ./public/  /usr/share/nginx/html/


 ---> 3956cff32507

Step 4/5 : EXPOSE 80


 ---> Running in b4c27124989d

Removing intermediate container b4c27124989d

 ---> ba9d1764d764

Step 5/5 : CMD ["nginx", "-g","daemon off;"]


 ---> Running in 61dca01a4883

Removing intermediate container 61dca01a4883

 ---> 2aadc5732a60

Successfully built 2aadc5732a60

Tagging built image with 192.168.26.56/library/blog:41
Docker Build Response : 2aadc5732a60
Pushing [192.168.26.56/library/blog:41]
The push refers to repository [192.168.26.56/library/blog]
89570901cdea: Preparing
65e1ea1dc98c: Preparing
88891187bdd7: Preparing
6e109f6c2f99: Preparing
0772cb25d5ca: Preparing
525950111558: Preparing
476baebdfbf7: Preparing
525950111558: Waiting
476baebdfbf7: Waiting
88891187bdd7: Layer already exists
6e109f6c2f99: Layer already exists
65e1ea1dc98c: Layer already exists
0772cb25d5ca: Layer already exists
89570901cdea: Pushing [>                                                  ]  301.6kB/28.75MB
89570901cdea: Pushing [==>                                                ]  1.193MB/28.75MB
476baebdfbf7: Layer already exists
525950111558: Layer already exists
89570901cdea: Pushing [======>                                            ]  3.917MB/28.75MB
89570901cdea: Pushing [==========>                                        ]  5.996MB/28.75MB
89570901cdea: Pushing [==============>                                    ]  8.097MB/28.75MB
89570901cdea: Pushing [==================>                                ]  10.76MB/28.75MB
89570901cdea: Pushing [=====================>                             ]  12.57MB/28.75MB
89570901cdea: Pushing [========================>                          ]   13.8MB/28.75MB
89570901cdea: Pushing [=========================>                         ]  14.71MB/28.75MB
89570901cdea: Pushing [===========================>                       ]  15.59MB/28.75MB
89570901cdea: Pushing [=============================>                     ]  16.79MB/28.75MB
89570901cdea: Pushing [===============================>                   ]  18.27MB/28.75MB
89570901cdea: Pushing [=================================>                 ]  19.45MB/28.75MB
89570901cdea: Pushing [===================================>               ]  20.34MB/28.75MB
89570901cdea: Pushing [=====================================>             ]  21.55MB/28.75MB
89570901cdea: Pushing [=======================================>           ]  22.44MB/28.75MB
89570901cdea: Pushing [=========================================>         ]  23.64MB/28.75MB
89570901cdea: Pushing [==========================================>        ]  24.52MB/28.75MB
89570901cdea: Pushing [============================================>      ]  25.42MB/28.75MB
89570901cdea: Pushing [==============================================>    ]  26.61MB/28.75MB
89570901cdea: Pushing [===============================================>   ]  27.19MB/28.75MB
89570901cdea: Pushing [=================================================> ]  28.69MB/28.75MB
89570901cdea: Pushing [==================================================>]  29.32MB
89570901cdea: Pushed
41: digest: sha256:c90b64945a8d063f7bcdcc39f00f91b6d83acafcd6b2ec6aba5b070474bafc37 size: 1782
Cleaning local images [2aadc5732a60]
Docker Build Done
[liruilong-cicd] $ /bin/sh -xe /tmp/jenkins246013519648603221.sh
+ export KUBECONFIG=/kc1
+ KUBECONFIG=/kc1
+ /kubectl set image deployment/nginxdep '*=192.168.26.56/library/blog:41' -n kube-system
deployment.apps/nginxdep image updated
Finished: SUCCESS
```

### 6、访问hexo博客系统

```bash
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$kubectl get deployments.apps  | grep nginxdep
nginxdep                  2/2     2            2           30h
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$kubectl get pods -o wide  | grep nginxdep
nginxdep-bddfd9b5f-94d88                             1/1     Running   0                 110s   10.244.171.142   vms82.liruilongs.github.io   <none>           <none>
nginxdep-bddfd9b5f-z57qc                             1/1     Running   0                 35m    10.244.171.177   vms82.liruilongs.github.io   <none>           <none>
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$kubectl get svc -o wide | grep nginxdep
nginxdep                            NodePort    10.106.217.50    <none>        8888:31964/TCP                 30h   app=nginx
┌──[root@vms81.liruilongs.github.io]-[~/ansible/k8s-deploy-create]
└─$kubectl describe  pods nginxdep-bddfd9b5f-94d88
Name:         nginxdep-bddfd9b5f-94d88
Namespace:    kube-system
Priority:     0
Node:         vms82.liruilongs.github.io/192.168.26.82
Start Time:   Fri, 04 Feb 2022 03:11:14 +0800
Labels:       app=nginx
              pod-template-hash=bddfd9b5f
Annotations:  cni.projectcalico.org/podIP: 10.244.171.142/32
              cni.projectcalico.org/podIPs: 10.244.171.142/32
Status:       Running
IP:           10.244.171.142
IPs:
  IP:           10.244.171.142
Controlled By:  ReplicaSet/nginxdep-bddfd9b5f
Containers:
  web:
    Container ID:   docker://669f48cb626d5067f40bb1aaa378268a7ee9879488b0b298a86271957c162316
    Image:          192.168.26.56/library/blog:41
    Image ID:       docker-pullable://192.168.26.56/library/blog@sha256:c90b64945a8d063f7bcdcc39f00f91b6d83acafcd6b2ec6aba5b070474bafc37
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Fri, 04 Feb 2022 03:11:15 +0800
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:        100m
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-trn5n (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-trn5n:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  4m10s  default-scheduler  Successfully assigned kube-system/nginxdep-bddfd9b5f-94d88 to vms82.liruilongs.github.io
  Normal  Pulling    4m9s   kubelet            Pulling image "192.168.26.56/library/blog:41"
  Normal  Pulled     4m9s   kubelet            Successfully pulled image "192.168.26.56/library/blog:41" in 67.814838ms
  Normal  Created    4m9s   kubelet            Created container web
  Normal  Started    4m9s   kubelet            Started container web
```

|                       访问hexo博客系统                       |
| :----------------------------------------------------------: |
| ![在这里插入图片描述](https://img2023.cnblogs.com/blog/3332572/202403/3332572-20240315133205914-50196778.png) |

