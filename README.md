# CICD-Gitlab-Jenkins-Docker-Harbor-K8S
Gitlab+Jenkins+Docker+Harbor+K8s集群搭建CICD平台(持续集成部署Hexo博客Demo)

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
