# drone-k8s部署

**特别提醒**

错误提示：

```
Login Failed. invalid character '<' looking for beginning of value
```

**使用1.x版本的drone配置旧版本的gitlab(8.8.X)时遇到了一个错误，最后更新了gitlab版本到`11.X`后才能正常回调到drone的登陆页面。**

权限设置

```
kubectl create clusterrolebinding --user system:serviceaccount:gitlab:default default-gitlab-sa-admin --clusterrole cluster-admin
```

> 经过网络搜索一翻，有这么一篇`[Gitlab配置Webhook，报错 Url Is Blocked Requests To The Local Network Are Not Allowed](http://zpycloud.com/archives/561)`
>
> Gitlab 10.6 版本以后为了安全，不允许向本地网络发送Webhook请求，如果想向本地网络发送Webhook请求，则需要使用管理员帐号登录.
>
> 即可进入Admin Area，在Admin Area中，在Settings标签下面，找到`OutBound Request`，勾选上`Allow Requests To The Local Network From Hooks And Services `，保存更改即可解决问题。
>
> 从GitLab 9.0开始，Prometheus及其导出器默认打开。
>
> 小团队，用的代码管理软件是gitlab，容器编排工具是Kubernetes建议用Gitlab-CI或Drone，开箱即用，可以减少很多工作量。
>
> 对插件有强烈需求，并且喜欢UI操作流水线的建议用Jenkins。

**关于权限**

- gitlab建议使用root用户授权，不然普通用户是没办法获取到项目信息。
- 如果使用普通用户授权了Developer加入项目，drone里能看到列表，但是不能编辑项目。



---

### Tekton和drone区别

> 采用Knative / Tekton的目标不是解决量问题本身，目标是利用由大量Kubernetes专家社区支持的解决方案。我相信这些技术将迅速发展并迅速改进; 这些项目拥有大量资源可供使用。
>
> Knative / Tekton意味着构建块，这意味着这些项目可以处理低级细节（卷，网络，命名空间，安全性），而Drone可以处理更高级别的细节（用户界面，权限，插件，版本控制集成）。这使核心无人机团队能够更专注。
>
> 因为Knative / Tekton是构建模块，没有高级集成，用户界面等，我认为这也是Drone填补空白并为Kubernetes社区增加价值的绝佳机会。
>
> see <https://github.com/drone/drone-runtime/issues/65>

### 遇到的坑：

- Drone的k8s版本只是试验阶段。生产建议使用docker方式安装
- 开启k8s或Nomad ，RUNNER 配置失效
- k8s的job存在过多，只能用`kubectl delete jobs --all` 手动删除，另外的job配置`ttlSecondsAfterFinished` 配置几秒后自动删除，目前无法配置。See <https://github.com/drone/drone-runtime/issues/69>

