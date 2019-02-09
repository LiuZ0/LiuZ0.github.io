---
layout: post
title: Kubernetes - kubelet bootstrap 流程
category: 技术
---

更新历史：

- 2018.09.18，初稿完成
- 2018.10.13，增加 kubeadm 的方法

很长时间没写东西了。离家在外两个人带娃很忙，在家空闲的时间基本都用来陪娃了，在加上前段时间在备考 CKA，时间上更是抠抠缩缩。业内人士都知道 CKA 是 Kubernetes(下面简称 k8s) 社区认证的管理员证书，我作为早期参与 openstack(下面简称 os) 的社区开发人员，openstack 的证书都没怎么关心过，现在为啥要考这个 CKA 呢？其实原因很简单，就是想对 k8s 多一些了解。我从2013年开始以开发人员的角色接触 os，当时年轻气盛，精力无限，一上来就是边阅读源码边安装试用，碰到问题都是通过读代码解决，从 os 内部的实现机制入手然后再从外往内看使用场景以及 os 的各种优势。时过境迁，人年龄大了，接触的东西多了，不能什么新事物和新技术一上来就是啃代码，没了年轻的资本，才不得不采取更为稳妥也更科学的学习方式，**从外到内，先会用，然后再了解其原理机制，必要时才去定制**。那怎么才算是会用呢，或者怎么才能向别人证明你会用呢？答案自然就是考证了。我见过有很多人是为了考证而考证，特别是那些急于找工作的。但 CKA 对我而言其实更多的是一种鞭策，让自己心里有个小目标，同时能够以运维人员而非开发者的身份去系统的学习使用 k8s。

## 参考文档

- [k8s 官网](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)
- 这位朋友的两篇博客都讲得不错，[文章一](https://mritd.me/2018/01/07/kubernetes-tls-bootstrapping-note/)， [文章二](https://mritd.me/2018/08/28/kubernetes-tls-bootstrapping-with-bootstrap-token/)
