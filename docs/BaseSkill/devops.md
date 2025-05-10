# ci/cd流程

通常的ci/cd worflow如下。

通常，CD有两种含义，第一种是持续交付，也就是将镜像版本构建好，但是不立即上线，最终上线由人工决定。第二种则是持续部署，也就是部署到生产环境。需要具体区分上下文语境，以及到底是研发团队还是部署实施团队。

> [浅谈持续集成(CI)、持续交付(CD)、持续部署(CD) - EdisonYao - 博客园 (cnblogs.com)](https://www.cnblogs.com/Sweettesting/p/14735920.html)、

> 通过server部署为例子，需要通过镜像来部署到集群，并且需要和其他服务一起统一部署prod/dev环境。

## 具体步骤

持续集成 & 持续交付(repo1)

- 开发代码。
- 提交pr，通过检测 & code review,合并到master。
- 由开发者自己控制，手动触发pipeline, 进行镜像的构建和推送。这里的CD指的是持续交付，也就是镜像版本已经准备好了。

此时镜像准备好了，可以进行prod / dev环境部署

持续集成 & 持续部署(repo2)

- 修改要上线的软件版本
- 提交pr,通过检测 & code review,合并到master
- 合并之后自动触发pipeline,将镜像并部署到prod/dev集群。
