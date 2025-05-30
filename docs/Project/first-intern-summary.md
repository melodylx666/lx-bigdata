# 第一段实习总结

## 岗位理解

第一段实习，是在一个自动驾驶独角兽公司的Data Infra岗。

其实对于这个岗位的实际定位，没入职之前我的理解是一直有偏差的。那时候我的理解就是进去之后搞各种开源组件，比如Spark,Flink等内核的二次开发，或者说可能是每天写SQL。但是入职之后，我才了解数据中台 | data Infra的具体定位，以及实际的岗位职能。数据中台就是要打破数据孤岛，统一管理以减少重复工作，提高数据的存储 & 计算效率。中台岗位不止有数据中台，也会有其他中台。简单来说，把它当做一个Project中的Common模块就可以。

Data Infra 岗必然是依赖于下游的业务团队(此处是算法团队)而生存的。所以显而易见，如果想要团队或者个人的发展有好的产出，必须有敏锐的业务嗅觉，能够识别每个阶段公司的核心算法团队，并及时沟通，了解痛点。痛点了解之后，需要了解其他团队是否已经有现存的工作，如果没有/有很大改进空间，那就可以考虑推出自己的新产品来填补空缺，并且一定要从项目设计书开始就时刻突出项目的所解决的核心问题。**如果自己不能说服自己，那就不要做。如果可以说服自己，那就坚定地做下去，见招拆招**。


## 技术体验

> Infra岗从前后台视角来看，往往都可以被划分到后台开发。

我进去的岗位语言以Python为主。我进去的任务就是开发一个通用的算法任务提交平台，规模挺大的。但是当时我对后台开发的理解很浅，微服务设计更是没有了解。但是mentor愿意给我机会，让我开始就上手一个特别复杂的任务，这个任务耗费了我小一个月时间去完成。这个项目是我投入精力最大，但是产出相对最小的项目。但是对个人而言它意义重大，因为**它是我使用过hack技术最多的需求**。我似乎把之前听过的80%的hack方案都使用了一遍。各种注册 | 缓存 | 元编程 | 代码解析 |进程日志匹配 技巧，每一步都是写方案时候没想到的，但是每一步都有它存在的意义。

后续的需求，对我而言是一种四两拨千金的感觉。因为**我可以通过一小段核心代码**，达到远超预期的收益。而这一段核心代码以及初版设计，我只花了一天的时间就完成了。后面的流程虽然还是有小插曲(因为对CICD不足够熟练)，但是最终这些需求还是完美完成了。

简单总结：语言不重要，思想很重要。永远要同时站在开发侧 & 用户侧(下游团队)两侧同时看待问题，基于痛点来设想方案，而不是沉浸在自己的代码世界里。

## 关系相处

mentor，以及leader，大家都友善，都愿意给机会，这点是我永远感激的。当时第一场面试的时候的唯一内容，就是让我手写B树并展开了关于它的所有知识点，我居然还是接下来并写了大体框架(没法也没能力完整实现，因为一个教学版本可能就需要700行，而工业级高效代码可能需要几个人搞一年时间)，现在想想当时的我也还是比较大胆的，但是他们也往往欣赏这种大胆，而不是跟随八股循规蹈矩。

## 遗憾

最终还是被不可控因素强制叫回学校了，但是我在一夜未眠之后还是控制情绪把公司这边的事情处理完了，也没有向朋友广而告之我离职的无奈。这对向来习惯抱怨的我来说是一种巨大的进步。

可能我永远都不会忘记晚上一个人下班后在学知园路旁，边走边哽咽的场景。它就是我这段路途的缩影，纵然荆棘丛生，风雨交加，但是我不能停止脚步，我只能继续向前。毕竟，“杀不死我的，只会使我更强大”。适应性强永远是我的第一标签，困难只会让我更加坚定这一信念。
