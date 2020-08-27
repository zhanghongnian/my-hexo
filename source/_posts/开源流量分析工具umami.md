---
title: 开源流量分析工具umami
date: 2020-08-25 22:43:44
thumbnail: /img/umami-preview.jpg
tags: 
- 流量分析
categories:
- 监控
toc: true
---

umami 是一款简单易用，美观的网络流量分析工具。目标是提供给你一个友好的，可私有化部署的开源工具替代需要第三方的服务，如[百度统计](https://tongji.baidu.com/web/welcome/login)。你可以在[这里](https://umami.zhnliving.cn/share/n5BMbCRy/my-hexo)看到本站的流量分析. 本文大部分是翻译[官方文档](https://umami.is/docs/about)。
<!-- more -->

# 特点
- umami 采集一些基本的网站流量分析指标指标：页面浏览量，操作系统，浏览器，和用户来源。
- 所有的指标都展示在一个简单易用的浏览器网页中。
- 可配置多个网站进行数据收集。
- 需加载的统计脚本很小（6kb以下）并且支持远古浏览器比如 IE。
- 如果你想公开你的统计，可以在系统中生成一个分享链接分享给其他人。
- 手机端友好，umami 的样式针对手机端优化过，你可以在手机端方便的查看相关信息。
- 关注隐私，umami 不收集任何个人敏感数据，所有收集上来的数据都进行了匿名处理
- 开源，umami 是开源的，使用 mit license，源代码在 [Github](https://github.com/mikecao/umami)。


# 开始使用
## 安装
本文采用在自己搭建的 minikube 上运行 umami。
1. 初始化依赖
{% codeblock "初始化依赖" lang:sh %}
# 下载项目
git clone https://github.com/mikecao/umami.git
# 进入项目
cd umami
# 导入库表结构
mysql -u username -p databasename < sql/schema.mysql.sql
{% endcodeblock %}
执行完毕后会创建一个本地账户，用户名 admin，密码 umami

2. 构建 docker 镜像
{% codeblock "构建 docker 镜像" lang:sh %}
# 打包
docker build --build-arg DATABASE_TYPE=mysql . -t zhnliving.tencentcloudcr.com/study/umami:v1.0.1
# 推送
docker push zhnliving.tencentcloudcr.com/study/umami:v1.0.1
{% endcodeblock %}
注意 `--build-arg DATABASE_TYPE=mysql` 一定要写，否则会按默认配置打包成 Postgresql 专用的镜像。
本人由于要部署到云上服务器，所以选择打包镜像到云上，有兴趣的小伙伴们也可以使用这个镜像地址。

3. 在 minikube 上部署 umami
{% codeblock "umami k8s 配置" lang:yaml >folded %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: umami
  namespace: default
spec:
  selector:
    matchLabels:
      app: umami
  template:
    metadata:
      labels:
        app: umami
    spec:
      containers:
      - name: umami
        image: zhnliving.tencentcloudcr.com/study/umami:v1.0.1
        env:
        - name: DATABASE_URL
          value: "mysql://username:password@mysql-host:3306/umami"
        - name: DATABASE_TYPE
          value: "mysql"
        - name: HASH_SALT
          value: "your-random-string"
        ports:
        - containerPort: 3000
          name: umami
---
apiVersion: v1
kind: Service
metadata:
  name: umami
  namespace: default
spec:
  type: NodePort
  ports:
    - port: 3000
  selector:
    app: umami
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: umami-ingress
  namespace: default
spec:
  tls:
  - hosts:
    - umami.zhnliving.cn
    secretName: secret-tls-umami
  rules:
  - host: umami.zhnliving.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: umami
          servicePort: 3000
{% endcodeblock %}

## 登陆
初始化sql会创建默认的管理员账号 admin（密码 umami），所以登陆后第一件事就是更改账号密码。
<img src="login.png" width="400px">

登陆后点击头部的 Settings。然后点击左侧菜单中的 Profile 然后点击修改密码，修改一个强有力的密码吧
{% asset_img change-password.png 修改密码 %}

## 添加监控网站
登陆 umami 后，点击头部的 settings，然后点击左侧的 websites ，然后点击 add website 按钮。
{% asset_img add-website.png 增加监控网站 %}
填充好下面的表单点击提交按钮即可。
<img src="add-website-form.png" width="400px">

Name 字段可以填任意字符串，用来标示网站。
Doamin 字段填写监控网站的域名，
Enable share URL 复选框如果勾选上的话，分享链接就可以使用了。

## 收集数据
添加完网站之后，点击 Get tracking code 按钮
{% asset_img get-tracking-code.png 获取 tracking 代码 %}
从弹出的窗口中，复制代码，粘贴到你的网站 `<head>` 部分中
{% asset_img tracking-code-form.png tracking 代码 %}
然后进入你的网站，你将在 dashbaord 中看到有数据产生。

## 开启共享链接
在网站属性中如果勾选了 `Enable share URL` 复选框，可以点击 `Share URL` 按钮获取分享链接
{% asset_img share-url.png 获取分享 url %}
从弹出的窗口中，复制链接地址给别人，就可以让别人不用登陆就可以看到数据了
{% asset_img share-url-form.png 获取分享 url %}


# TODO & 脑洞
<input type="checkbox" onclick="return false" >  替代不子蒜，支持网站 pv/uv 展示
<input type="checkbox" onclick="return false" >  支持中国地图细分展示

# 参考
https://umami.is/docs/about