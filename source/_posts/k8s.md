---
title: Kubernetes
date: 2018-02-03 16:35:11
categories: DevOps
---
## CI/CD和DevOps

![](/images/k_DevOps.png)
图片来自[CloudBees](https://www.cloudbees.com).

<!-- more -->

+ 持续集成（Continuous Integration）的目的是让产品可以快速迭代，同时还能保持高质量。核心措施是代码集成到主干之前，必须通过自动化测试。只要有一个测试用例失败，就不能集成。优点就是能快速发现错误，防止分支大幅偏离主干。   

+ 持续交付（Continuous Delivery）是指频繁的将软件的新版本交付给 QA或用户 以供评审。如果评审通过，代码就进入生产阶段。持续交付可以看作持续集成的下一步。它强调的是，不管怎么更新软件是随时随地可以交付的。

+ 持续部署（Continuous Deployment）是持续交付的下一步，指的是代码通过评审以后，自动部署到生产环境。持续部署的目标是，代码在任何时刻都是可部署的，可以进入生产阶段。持续部署的前提是能自动化完成测试、构建、部署等步骤。

持续部署与持续交付的区别如下图所示：[- - -摘自阮一峰老师日志](http://www.ruanyifeng.com/blog/2015/09/continuous-integration.html)
![](/images/k_CD.png)
