# 2022年终总结


# 2022流水账式年终总结

## 前言

2022，是我大学生活中最为值得记录的一年，虽然也有许多坎坷和挫折，并不是一帆风顺的，但是还是想记录一下，希望以后能通过这篇文章来回顾我的2022。

## 职业发展

今年其实是在职业发展上变化最大的一年，经历了学习、实习和开源等事情后，对自己的职业道路渐渐也有了清晰的认知，不再是去年的盲目自大，现在更多是以一种低期望的视角去审视自己未来的发展。

### 学习

#### MIT6.824

今年的学习经历可以说是彻底把学校方面的学习抛开了，基本处于自学的状态。以一种极不自律的方式浅学了一些分布式方向和数据库方向的知识。  

21年年底，当时西安疫情严重，我们都被封在宿舍，舍友问我："要不要一起做一做MIT6.824？"，当时我还是一一个只会crud的java boy，只做了一些工作室的项目。当舍友问我的时候，我一开始还是没有什么兴趣的，因为我又不会go，还不懂分布式系统。然而那几天，我仔细审视了一下自己的道路发展，感觉如果一直做一些简单的接口和项目，对自己的未来发展并不会有很大的作用。于是答应了舍友，一起去看看这个lab。

刚开始学824的过程还是比较辛苦的，第一次看英文论文，第一次翻译论文，第一次学习使用Golang... 经过一段时间的摸索和讨论，直到2月多我们也渐渐的完成了这个lab。lab2的debug地狱让我明白打清楚日志是一件多么重要的事情，lab3和lab4也让我对整体的分布式系统设计有了一些基本的思路。

![91E0EEC3-81CC-4DF5-A388-C0D6ED5523FD_1_201_a](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/91E0EEC3-81CC-4DF5-A388-C0D6ED5523FD_1_201_a.jpeg "在宿舍看论文")

整体来看，这次的学习是一次十分有用的实践，让我从0开始入门了分布式系统，也开始对分布式系统的开发有了很浓的兴趣，虽然现在回头看，其实对于coding层面并不是一个非常大的提升，但是对于宏观层面的系统思考，确实启发了很多思路。

![6BD69320-C82C-4723-922A-F7A82151E65F_1_102_o](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/6BD69320-C82C-4723-922A-F7A82151E65F_1_102_o.jpeg "通过1024次测试")

#### TinyTxn

4月多的时候，得知PingCAP又办了一个TinyTxn训练营，也就是实现一个Percolator分布式事务算法，其实也就是TinyKV的lab4，本身就遗憾错过了去年的TinyKV训练营，这次一听到消息就报名了。

实际训练营的内容并不多，主要就是了解了Percolator分布式事务算法，完成了TinyKV背景下的分布式事物的实现，整体两天不到就完成了，但是也收获了很多分布式事物相关的知识(虽然现在也忘完了)。

![20230101143336](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101143336.png "混个小证书")

#### TinyKV

6月的时候，PingCAP举办了2022年的TinyKV训练营，当时我和峰哥也是很激动，因为本身就很遗憾去年没有参加，再加上对TinyKV本身也很感兴趣，我们就果断的报名了。

但是十分遗憾的就是当时刚好是期末复习周，再加上期末考后还有军训，基本上前一个多月没有时间去做这件事，导致我们刚开赛就将期望从拿名次调整成按时完成了。

可惜后来还是因为7月多之后，我和峰哥都去实习了，时间也只有周末了，众所周知，上班了周末是根本不想动了，导致我们再次将期望调整成早日完成。

在断断续续的commit后，终于在9月多基本完成了这个项目。整体来说，我认为它和6.824是一个类型的lab，但是又完全不同，TinyKV主要更重视状态机上层的KV实现和整体请求响应处理流程，824主要是更重视在Raft协议这一层。我认为两个都有做的必要，但是一定要一次性完成，不要像我一样，断断续续做了好几个月，其实都丧失了很大一部分的激情了，而且总是无法从一个准确的宏观角度去思考这个项目(因为做了后面，又忘了前面)。

总结来讲，这次的lab属于一次十分失败的实践，我和峰哥都没有投入足够的精力在这上面，导致TinyKV做是做了，但是很多地方又不是很懂。

![20230101143617](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101143617.png "才发现代码量竟然这么少")

#### MiniOB

9月多的时候，hzh问我要不要一起去参加MiniOB，当时这个比赛很火，已经听说了很多人去报名了，包括很多朋友也是早早的组好了队，我其实之前就拒绝过几次组队的邀请，主要原因是我实在不会也不想学cpp。我对于cpp一直是十分抵触的心理，任何和cpp沾边的事情我都不想去看。但是这次hzh说不会cpp没事，对于语言要求不高，再加上想着那就趁此机会尝试走出舒适圈，接触一下cpp吧。于是就答应了一起组队，名字就延续了hzh一直一来的队名：**奥特曼都是假的**。

在10月多开赛的时候，我们也早早的分工好了，一开赛就开始做。

![20230101143755](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101143755.png "先做了一些简单的任务")

也在比赛前期有着还不错的名次，但随着比赛的进行，以及今年的题目和往年极度的相似，也出现了很多直接冲到前面的队伍，当然也是有很多大佬开始发力了。导致后来名次一度降到了30名左右。

![20230101143841](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101143841.png "刚开赛前几天的名次还是挺满意的")

再之后，就因为队友一些秋招的事，以及自己当时也是时候开始准备下一段实习的面试了，就决定这个比赛暂时放弃了，虽然很可惜，但是后来我们队伍又在一起做了别的事情，这次的比赛也就没有那么的遗憾了。

整体来说，这次的比赛也算我第一次接触到db内核，虽然是一个简单的db，但是也让我对db的底层更加了清晰了，也尝试动手写了一些赛题，也算浅浅入门了db吧。

---

### 实习

实习算是我今年最重要的事情，从年初的准备实习面试到年末的准备实习面试，算是贯穿了2022年的一条主线。
4月多在学长的内推下，经历了三轮技术面一轮Hr面以及长达大半个月的等待后，成功面进了当时的梦想公司，在当时也算是实现了我从学习技术以来的一个小目标。

![20230101143956](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101143956.png "第一次看到还是很激动的")

在面完之后，并开始了长达3个月的摆烂生活，直到7月多，在兜兜转转的行程中到达了首都。

![20230101144102](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101144102.png "走的太急，老贵了")

在北京吃了第一顿便饭，感概北京物价的同时，也找到了比我早一个月去实习的学长，开始畅想着美妙的实习生活。

![41545A48-345E-477C-9F07-AC8ABE534436_1_201_a](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/41545A48-345E-477C-9F07-AC8ABE534436_1_201_a.jpeg "这要40？？")

当晚就迫不及待的去公司楼下看了眼，在很多学长和朋友的朋友圈已经看过很多遍了，但是第一次现实看到，还是有几分感概的，感概自己算是完成了一个小心愿，让我在这条职业道路上终于看到了些正反馈。

![042ED75E-86F2-41BE-9842-1C02A6B7DB7B_1_102_o](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/042ED75E-86F2-41BE-9842-1C02A6B7DB7B_1_102_o.jpeg "这个点还是灯火通明")

第二天就开始去找房租房，在7月酷暑之下，奔波了一整天，终于有了个合适的选择，虽然又贵又旧，但是总归有个落脚处了。也发了个朋友圈，纪念一下正式北漂的第一天。

![20230101144436](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101144436.png "2022年唯一一条朋友圈")

之后又接到通知，说珠海市是风险地区，得居家隔离5天，在家枯燥了5天之后，终于正式开始了实习生活了。

别的不说，字节的伙食确实是十分可以的，也一度让我体重飙至新高。

![20230101144531](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101144531.png "每天的盼头就是吃饭")

在开始实习后，组里做的是字节的消息队列业务，带我的mentor是做内部一个基于mq的数据传输系统的，基本上我这段实习都是在做这个数据传输系统的一些开发工作。

这个系统差不多是mentor一个人搭建起来了，而且再加上我来之后，mentor也搬到我旁边，很快就了解了这个系统。但是一开始也并没有做一些写代码的工作，也是做一些服务上线以及一些比较边缘的工作。在前一个多月，基本没有写过什么功能。

之后开始慢慢写一些小需求，从简单的服务接口到系统的优化都有接触到，总体下来，其实主要更多对java的编写更熟悉了一点，虽然组里是做mq的，但是我并没有什么机会接触到mq。基本是在和这个数据传输系统打交道，期间也是做了一些比较不打杂的工作，比如一些数据assign不均、系统节点流量限制等特性。即使代码量不大，但是也算是做了一些微不足道的工作。

后来在10月多，因为学校那边事情很多，再加上房租三个月刚好到期，而且自己其实想再去看看别的方向，比如流计算和大数据引擎开发等方向，于是和mentor和leader提了离职。

也十分感谢mentor三个月的耐心指导，工作中遇到的问题也都得到了mentor的很多帮助。最后在临走前两天，mentor也请我吃了顿饭(十分感激。

后来回想这段实习，和同事们也没有相处的很融入，除了mentor其他同事基本上也就是打个招呼的关系，可能是大家和我不是一个年龄段的，也可能是因为自己还是不适应在职场社交，导致最后也没有在这段工作中和同事们成为朋友。这也算是在这次实习后期经常emo的原因之一吧。

总结来看这段差不多三个月的实习经历，成长是有的，对于一些工具的使用和排查日志技巧也提升了不少，但是对于分布式系统和存储这方面，其实并没有很大的提升，也主要是因为工作并没有接触到这些，当时进来的时候，职位是分布式存储实习生，但是实际工作并没有接触到这些，也算是有一定的失落，虽然工作很轻松，mentor很好，但是还是没有达到实习前对自己的期望。

---

### 开源

今年第一次接触到了开源，也在这个方向进行了许多尝试，包括去参加很多开源夏令营，以及自己去社区深入交流，当然，开源也是我今年最有意义但又最遗憾的事情了。

今年参加了OSPP、ASOC和GLCC的开源夏令营，分别在DLedger、Dubbo和Seata社区做了一些微不足道的贡献，是第一次尝试去完成开源社区的任务，即使任务也都不难，我也算是借此契机进入到了开源世界。在和朋友的交流中，愈发认为开源十分重要，甚至可以说比实习还要有意义。

可惜的就是我虽然这些项目也顺利结项了，但是任务不是很难，也不是很核心，代码量也不大。因此其实并没有机会在社区更加深入的参与，甚至Dubbo和Seata我就没有参加到社区活动中，而且由于任务并不难，所以其实提升并没有很大。再加上因为后期时间安排问题，Dubbo那边基本没有足够的精力投入进去，导致任务完成度很低，甚至可以说是烂尾，也很对不起项目导师远云。所以也是留下了许多遗憾。

![20230101144925](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101144925.png "OSPP")

![20230101145202](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101145202.png "GLCC")

![20230101145246](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101145246.png "ASOC")

除了参加了这些开源夏令营活动之外，我还自己尝试去社区做一些贡献，一开始由于实习是在mq的组，因此选择去RocketMQ社区做一些贡献，在社区导师的帮助下，也贡献了几个小的pr。虽然并没有做很多贡献，但是算是我开源之路的一个里程碑了，尝试自己去社区做贡献和参加开源活动做贡献是不太一样的。如果社区没有人愿意带你，是很难自己去融入到社区，因为很多任务也不会assign到你身上。这在之后我尝试去别的社区做贡献的时候也感受明显。

当时也尝试去Pulsar、IoTDB等社区做贡献，后来发现基本任务也都是商业化公司内部的人在做，而且也很难找到人愿意带，因此自己很难有一个起点去开始社区贡献。后来也是感受不到太大的正反馈，因此也暂时告一段落了。
今年的开源只能说入了门，核心的事情都没有接触到，而且今年下半年属于摆了一波大烂，甚至出现了几个月没上过github的情况，真是越想越惭愧。

> 不过还是有在一些社区混到了小奖品

![1D379BC5-6ADE-4888-BBC1-1B96796428EB_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/1D379BC5-6ADE-4888-BBC1-1B96796428EB_1_105_c.jpeg "Matrix Origin发的数据库少女？？")

![B575B432-15FD-47BC-AC73-D9B427F3DC71_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/B575B432-15FD-47BC-AC73-D9B427F3DC71_1_105_c.jpeg "P社的保温杯")

![20230101145832](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101145832.png "属于是摆烂式参与开源了")

希望明年能更专注到在某个社区进行贡献，希望能拿一个**commiter**。

---

## 生活

### 流水账

今年的生活还是一如往年的枯燥，简单记录一下我平凡生活中的琐事吧。

就流水账记录记录，基于手机照片，低程度还原我的枯燥人生。
> 1月

年初西安疫情原因，在学校被封了一个多月。

每天除了做核酸，就是在宿舍看剧，当时把风骚律师和硅谷基本都看完了，又重温了一遍绝命毒师，不得不说，看剧真的会上瘾。

终于学校让回去了，于是就大早上冒着小雪踏上了回家的路。

![6746C262-5E08-461F-B617-082BDCBDEE70_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/6746C262-5E08-461F-B617-082BDCBDEE70_1_105_c.jpeg "凌晨4点的校园")

> 2月

回家之后也和和朋友们聚会和打球，在被封一个多月之后，感受到能出门活动是一件多么幸福的事情。
![20230101150140](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101150140.png "好球配好背景板")

![20230101150236](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101150236.png "儿童框杀手")

在家过年也是把年前立的flag忘的一干二净，年前说要多看面经，多写博客，最后啥也没干就光玩了。

不过回想一下，感觉以后真正工作了，假期也是越来越少了，趁还在上学，假期还是得多玩玩啊。

> 3月~4月

这两个月基本上都在等待面试和等待面试结果，啥也没干，既枯燥又无聊，每天就睡到中午，去工作室坐坐，看看b站，一天就过去了。太消磨意志辣！

> 5月

面试结果出来了，暑假去向也是确定了，于是就和朋友们偶尔出去玩玩，想到自己其实来西安两年了，也没咋在西安溜达过，就也出去转了几次。

![F3278150-70C8-4E92-ADA7-5750FD3C6E52_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/F3278150-70C8-4E92-ADA7-5750FD3C6E52_1_105_c.jpeg "西安城墙当街溜子")

![DA31D3C7-C8B0-48D1-9C87-4BE7F6FBCB15_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/DA31D3C7-C8B0-48D1-9C87-4BE7F6FBCB15_1_105_c.jpeg "烂怂大雁塔")

![3ABA6418-BE90-489E-B6DB-B68AC450A3CE_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/3ABA6418-BE90-489E-B6DB-B68AC450A3CE_1_105_c.jpeg "大金链子小手表，一天三顿小烧烤")

![E5C539BC-D003-4F31-8FCA-E0BBFAB565B3_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/E5C539BC-D003-4F31-8FCA-E0BBFAB565B3_1_105_c.jpeg "困了(绝不是醉了)")

> 6月-7月

这个月主要是期末复习，由于又是一个学期没学，又开始了经典的工作室通宵复习(虽然5点多就起床开始打游戏了)。

月底又开始了很憨的军训，教官和我们同龄，所以也都很轻松，感觉每天溜达几步，然后树荫地下坐一天，差不多就过去了。

后来疫情原因，各种传西安马上要封城，然后学校也立马下通知了，让我们提前结束军训了，这时候我们也担心晚了走不掉了。就在当天下午两小时内就从决定回家到出校门了。

因为暑假要去北京实习，于是就打算直接去北京，北京的健康宝的弹窗可以评为新时代最sb的设计之一，导致我不得不改变行程，先去咸阳待了一晚，然后回珠海刷新我的行程，等待弹窗解除。

![0DFFDAD7-D98F-41C1-B866-0FF3AB33394C](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/0DFFDAD7-D98F-41C1-B866-0FF3AB33394C.jpeg "和咸阳地头🐎吃点串")

![20230101151123](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101151123.png "咸阳人半夜真不睡啊")

在回家待了一周之后，终于在13号凌晨看到了弹窗解除，当机立断，直接买了下午的高价机票，不想再因为疫情原因耽误行程了。

![49122BDC-DF9C-4F24-8968-B758A6717956_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/49122BDC-DF9C-4F24-8968-B758A6717956_1_105_c.jpeg "第一次来大兴机场")

之后就是居家隔离5天。

![DB852DF4-2E93-442E-B7EE-DD161ED20AE4_1_201_a](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/DB852DF4-2E93-442E-B7EE-DD161ED20AE4_1_201_a.jpeg "真吃了5天火鸡面")

然后就开始了实习生活了。

> 8月

在来北京之前，看了很多旅游攻略，总觉得自己既然来了，肯定要多去北京里面转转，这其实和我来西安上学之前是一样的心理，但是其实来了就不想到处看了。

实习的生活两点一线，基本没有变化，平平淡淡的，也只有偶尔出去看看首都的繁华，当然，用一句话来说就是：*海淀不是没有夜生活，而是海淀的夜生活与你无关*。作为一个小地方来的北漂实习生，在北京的感受就是孤单，特别是没有很多认识的朋友，同事们也不熟，导致自己长期处于一个社交孤立的状态，再加上工作的疲惫，一度产生离开北京的想法，在10月多更是强烈。

![67C189A6-5A9E-490F-9D38-E3E5FD12E22D_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/67C189A6-5A9E-490F-9D38-E3E5FD12E22D_1_105_c.jpeg "来北京还是得去故宫溜达的")

周末偶尔去公司蹭蹭工位的4K屏看电影，还有免费的小零食

![8DA9A3C8-5BEC-4F73-840C-C91BE9A72B36_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/8DA9A3C8-5BEC-4F73-840C-C91BE9A72B36_1_105_c.jpeg "憨豆的黄金周")

平时工作日，基本就是和学长一起吃饭，然后每天晚上吃完饭，都会在楼下溜达半小时，看看天，畅聊一下未来。

![55BF2C87-B95C-4702-8EC9-5A0BBA731897_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/55BF2C87-B95C-4702-8EC9-5A0BBA731897_1_105_c.jpeg "某天晚饭后的夕阳")

在网上老是刷到长安街骑行，于是和学长一起去在长安街骑行了一次。人很多，但是大家看起来都很开心，都相信明天会更好。

![20230101151846](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101151846.png "长安街的骑行大军")

![F6DE499E-4A6C-457E-A458-D464CD824100_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/F6DE499E-4A6C-457E-A458-D464CD824100_1_105_c.jpeg "第一次现实中见到天安门")

打球的人应该都知道东单篮球场，一直被称为中国街球圣地，于是我就也去东单感受一下最纯正的中国街球文化。

![DD8786AC-1FA0-496D-88EB-92FF8AFD970E_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/DD8786AC-1FA0-496D-88EB-92FF8AFD970E_1_105_c.jpeg "东单")

![20230101152028](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101152028.png "感受国内顶级扣将")

> 9月

和学长一起去了天坛溜达。可惜是白天去的，晚上去的话，可以看到天坛灯光秀。

![D8471418-9312-45C4-8C28-130B8D88D9A3_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/D8471418-9312-45C4-8C28-130B8D88D9A3_1_105_c.jpeg "祈年殿一角")

![B7C42EDD-CF6A-44FD-8E01-907F77AC591F_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/B7C42EDD-CF6A-44FD-8E01-907F77AC591F_1_105_c.jpeg "天气不错")

实习的时候，公司配备的Macbook Pro，在用习惯MacOS之后，真的是不想用回windows，于是刚好实习也靠自己攒了点钱，咬咬牙(牙都咬碎了)，买了人生中第一台mbp。

拿到之后，刚好公司搬工位，就和我的设备们一起合了个影。

![7F2658DE-5DCB-4E39-9E80-9899C1945E74_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/7F2658DE-5DCB-4E39-9E80-9899C1945E74_1_105_c.jpeg "差生文具多")

> 10月

这个月中旬结束了在字节的实习，又是由于疫情原因，学校不让返校，只能回家过渡过渡。

这次回家也算开始自己学着做饭吃，虽然不算很好吃，但是也算能喂饱自己了。

![20230101152450](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101152450.png "能吃就行")

在家的时间过的很快，除了没学习，其他都没落下，又是打球又是追剧的，可以说是这一年感觉最轻松的时候了。

> 11月

这个月终于接到消息可以回学校了，于是就踏上了回学校的路。

![AC735443-DE5A-43E0-B2BF-FAB8CA212906_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/AC735443-DE5A-43E0-B2BF-FAB8CA212906_1_105_c.jpeg "久违的西安地铁")

回学校之后最麻烦的就是还得先隔离三天，三天基本上就是看剧度过，不然真的不知道该干啥了。

![1CAAA381-4130-41AD-AA2E-10E4A06F767B_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/1CAAA381-4130-41AD-AA2E-10E4A06F767B_1_105_c.jpeg "隔离环境一般")

在学校也就开始准备了下一次实习的面试，又是字节，但是这次流程走的很快，面试都通过了，于是我以为稳了(后续证明赌狗真该死啊)，就开始了网瘾少年的生活。

和lzc打csgo双排了大半个月，每天从早打到晚，还有一次打了整整20小时，打了一个通宵，第二天早上做了个核酸就回去睡了。

![20230101152802](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101152802.png "早上已经要昏迷了")

后来又和朋友们一起玩求生之路。

感觉很久没有这种玩游戏的感觉了，上一次有这种游戏瘾的感觉还是初三打王者的时候。

这个月基本上就是在打游戏中度过了，虽然有趣，但是回想起来，又是满满的惭愧啊！

> 12月

有一天做核酸，从出宿舍门开始，手机就一直重启，直到出示健康码的一刻，才正常开机了，实在受不了这个气啊，一怒之下，又咬了咬牙(假牙都咬碎了)，打算给自己换了台新手机，再加上之前实习也存了点，就想着给自己换个好点的，希望能用久点吧。

![71DFFFC7-6826-4CAC-960B-F3AED503D6B4_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/71DFFFC7-6826-4CAC-960B-F3AED503D6B4_1_105_c.jpeg "就是心疼")

后来学校通知可以提前离校了，但是我实习还没正式确定下来，虽然上个月面试流程都走完了，但是offer一直没发，于是就一时兴起，打算一边去玩一边等结果。

之前看过一部电影，叫《心花路放》，所以我在产生想去玩的念头的时候，第一个想到的就是去大理。于是和峰哥商量一下，他也很想去，在他接到他的面试结果的一刻，我们立马决定去大理了，为了防止自己后悔，直接买好了机票。

可恶的是，晚上我就得知了因为公司hc盘点问题，没有hc了，我这次实习被鸽了。

![20230101153059](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101153059.png "永远不要all in")

在经历了几小时的心态调整后，还是决定继续去玩一趟吧(其实主要是机票已经买了，退了也太亏咯)，就当散散心了。

于是我们就踏上了为期一周的云南旅程。

![58F74108-742E-4540-94EA-89031CA1532A_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/58F74108-742E-4540-94EA-89031CA1532A_1_105_c.jpeg "西安 -> 昆明")

飞机到达昆明之后，因为没有打算在昆明玩，于是当天直接高铁去大理。

![76851986-2A6A-4B3B-AF54-A04F9B927F28_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/76851986-2A6A-4B3B-AF54-A04F9B927F28_1_105_c.jpeg "到达的一刻还是觉得太不现实了")

路上听着郝云的《去大理》，"*也许爱情就在洱海边*"。

![CC1DD196-A38E-440C-852B-E8E7729D77A6_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/CC1DD196-A38E-440C-852B-E8E7729D77A6_1_105_c.jpeg "入住大理古城里的民宿")

![2C26942E-2D27-4389-99B1-68FCBC56ED38_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/2C26942E-2D27-4389-99B1-68FCBC56ED38_1_105_c.jpeg "老板可爱的萨摩耶")

我们选择环游一圈洱海，整整140公里！到最后屁股都坐不住了，不过是很值的一圈！一路经过海西龙龛码头、喜洲古镇、海东双廊、小普陀等地方，一次把洱海一圈看了个够。

![86FFBCDC-F8A5-4F92-B94B-73E5DFE2D51D_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/86FFBCDC-F8A5-4F92-B94B-73E5DFE2D51D_1_105_c.jpeg "提辆敞篷")

先去了龙龛码头，这里很多🦆，还有很多在拍婚纱照的，我想如果我有机会，我也想来这里拍！

![20230101153942](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101153942.png "龙龛码头")

![566EC218-978B-4AE8-B569-D00913C090FF_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/566EC218-978B-4AE8-B569-D00913C090FF_1_105_c.jpeg "这比海更海")

![AA80C427-79C2-46E9-811C-6E925A10CBD2_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/AA80C427-79C2-46E9-811C-6E925A10CBD2_1_105_c.jpeg "祝福这对新人！")

![CB6C0245-78FF-447E-8B0F-D2A3CED9EDEF_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/CB6C0245-78FF-447E-8B0F-D2A3CED9EDEF_1_105_c.jpeg "苍山洱海之间")

中午我们又赶去了喜洲古镇，时间原因并没有细转，如果以后有机会还会来这里认真转转的。

![CC6E8C9E-3DBD-4B30-BB74-1AF876C332F6_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/CC6E8C9E-3DBD-4B30-BB74-1AF876C332F6_1_105_c.jpeg "转角楼")

![35E050C5-0EF7-4A3C-988E-244FE9A72A7A_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/35E050C5-0EF7-4A3C-988E-244FE9A72A7A_1_105_c.jpeg "海天一色")

![0ED3A463-66B6-4AE9-97B9-FABA8F718A0D_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/0ED3A463-66B6-4AE9-97B9-FABA8F718A0D_1_105_c.jpeg "洱海旁发呆")

![50906501-DED6-48FA-8BF5-4B56168BCAD7_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/50906501-DED6-48FA-8BF5-4B56168BCAD7_1_105_c.jpeg "午后洱海")

![6F25AFE8-9774-42A5-8CCD-BF89B7510267_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/6F25AFE8-9774-42A5-8CCD-BF89B7510267_1_105_c.jpeg "通往未知的道路")

![AF757064-2F5B-45BF-B147-B6C77E88122E_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/AF757064-2F5B-45BF-B147-B6C77E88122E_1_105_c.jpeg "忘了带🍟")

![B3B1858D-B89D-44DC-B0A0-673C47E34F0D_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/B3B1858D-B89D-44DC-B0A0-673C47E34F0D_1_105_c.jpeg "夕阳下的洱海")

洱海是一个很美的地方，我虽然在珠海长大，也算是在海边长大的，但是我感觉洱海比大海给我带来的视觉感受更震撼。

之后我们又出发去了丽江。

我们先去了玉龙雪山。因为山上没下雪，也没什么积雪，所以其实时机不太对，但是*来都来了~*

![6B299D66-5361-4811-8412-FFA1B62BA1EB_1_102_o](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/6B299D66-5361-4811-8412-FFA1B62BA1EB_1_102_o.jpeg "小🐮也会有烦恼吗")

![108EACAC-3519-4E26-AD19-1967E314976F_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/108EACAC-3519-4E26-AD19-1967E314976F_1_105_c.jpeg "玉龙矿山")

从平台爬到4680的路过于漫长了，有些高反的我，在此行之后开始重新思考下次去西藏是否顶得住了。

![DF6ABBD4-40AE-45F2-BBB6-E7C9B07ABA5D_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/DF6ABBD4-40AE-45F2-BBB6-E7C9B07ABA5D_1_105_c.jpeg "真顶不住了哇")

![20230101155500](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/20230101155500.png "4680打个卡")

下了雪山之后，我们来到了我觉得比雪山更美的地方-蓝月谷。

![2E2F57A5-9453-4FB4-A87C-6842EC53F5CC_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/2E2F57A5-9453-4FB4-A87C-6842EC53F5CC_1_105_c.jpeg)

![AABF59F9-CBDC-4290-9FAB-C0217D0F5FA7_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/AABF59F9-CBDC-4290-9FAB-C0217D0F5FA7_1_105_c.jpeg "🐮哥排泄物有点臭")

![89A914AC-FA11-4D87-AA85-A7595F454390_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/89A914AC-FA11-4D87-AA85-A7595F454390_1_105_c.jpeg "要是雪山再多点雪就好了")

![A5AE085A-65BA-4404-BE63-F68FFEB3544C_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/A5AE085A-65BA-4404-BE63-F68FFEB3544C_1_105_c.jpeg)

![225FFE59-B402-40D7-A767-B6256EE65610_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/225FFE59-B402-40D7-A767-B6256EE65610_1_105_c.jpeg )

结束之后，由于峰哥发烧了，再加上旅店老板说这个季节不适合去香格里拉，于是香格里拉就暂时放弃了。

之后我们还看了世界杯决赛，全程仰卧起坐，恭喜阿根廷！恭喜梅西！

在云南待了一周之后，现在回家，开启了每日追剧的生活，看了《机智的监狱生活》，《想见你》和《回来的女儿》。强推前面俩，后面那个高开低走，太可惜了。

**今年基本上也就是这么一个枯燥的开始和枯燥的结束，我前面20年何尝不也是呢？**

---

### 平台年终总结

#### 音乐

![4C21DD56-FE33-46CC-947C-5042AF6E072A_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/4C21DD56-FE33-46CC-947C-5042AF6E072A_1_105_c.jpeg)

![5A3D6D8D-037F-4745-A8A7-32FBB6FC24D8_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/5A3D6D8D-037F-4745-A8A7-32FBB6FC24D8_1_105_c.jpeg "我没有擦去争吵的橡皮，只有一枝画着孤独的笔")

![B54F2143-2553-47DD-8FFD-06AFD595A35C_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/B54F2143-2553-47DD-8FFD-06AFD595A35C_1_105_c.jpeg "周王陶！")

毫无疑问，年度歌手还是jay，陶喆上半年基本听了整整半年，不得不说周王陶真的是华语乐坛天花板了。

#### 影视

今年还是看了不少电影和剧的，有些忘了在豆瓣上标注了。在北京实习一个人孤单的时候，还靠看剧让我度过了那段时光。

![E9C95C5D-4E7D-4DA5-A50B-C622554FFA0E_1_105_c](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/blog/2023-01-01/E9C95C5D-4E7D-4DA5-A50B-C622554FFA0E_1_105_c.jpeg)

---

## 收获、遗憾与展望

### 收获

- 拥有了一次实习经历
- 第一次参与到开源世界
- 认真思考了自己的未来发展
- 大学以来第一次去旅游
- 靠自己工作买了很多想买的东西

### 遗憾

- 工作以打杂居多
- 开源没有持之以恒，做的东西都不够有含金量
- 参加了很多比赛，但是都没有投入足够精力
- 身材管理失败
- 还是单身
- 对于很多定下的目标，放弃的太快

### 展望

- 找到满意的暑期实习
- 秋招顺利，找到满意的工作
- 在开源社区做出一些有意义的贡献
- 成为一个项目的Commiter
- 身材管理下来
- 多练球、多打球
- 谈恋爱
- 看几场演唱会
- 秋招后去川藏线自驾
- 攒钱，毕业前去一趟北欧
  
