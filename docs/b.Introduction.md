# 介绍


## <font color='red'>**我们的设计为什么会出错**</font>

当你听到“混乱”这个词时，你会想到什么？也许你会想到喧闹的证券交易所，或者早上厨房里的一切都杂乱无章。当你想到“秩序”这个词时，也许你会想到一个空荡荡的房间，宁静而平静。然而，对于科学家来说，混乱的特点是同质性（相同性），秩序的特点是复杂性（差异性）。

例如，一个精心照料的花园是一个高度有序的系统。园丁用小路和栅栏划定边界，并划出花坛或菜地。随着时间的推移，花园会不断发展，变得越来越茂密；但如果不刻意耕耘，花园就会变得荒芜。杂草和草会扼杀其他植物，覆盖小路，直到最后每个部分都恢复原样——荒芜和无人照料。

软件系统也容易陷入混乱。当我们第一次开始构建一个新系统时，我们怀着宏伟的设想，认为我们的代码将是干净且井然有序的，但随着时间的推移，我们发现它积累了繁琐和边缘情况，最终变成了一堆令人困惑的管理类和实用模块。我们发现我们合理的分层架构已经像一块湿透的琐事一样崩溃了。混乱的软件系统的特点是功能千篇一律：API 处理程序具有领域知识并发送电子邮件和执行日志记录；“业务逻辑”类不执行任何计算但执行 I/O；所有东西都与其他东西耦合在一起，因此更改系统的任何部分都充满危险。这种情况非常普遍，软件工程师有自己的术语来描述混乱：大泥球反模式（现实生活中的依赖关系图（来源：Alex Papadimoulis 的“企业依赖关系：大泥球”））。

<figure markdown='span'>
    ![apwp 0001](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0001.png){ align=center }
    <figcaption><font size=2>图1.现实生活中的依赖关系图（来源：Alex Papadimoulis 的“企业依赖关系：大泥球”）</font></figcaption>
</figure>

!!! tip
    泥球是软件的自然状态，就像荒野是花园的自然状态一样。需要能量和方向来防止软件崩溃。

幸运的是，避免形成大泥球的技术并不复杂。


## <font color='red'>封装和抽象</font>

封装和抽象是我们作为程序员本能地使用的工具，即使我们并不都使用这些确切的词语。请允许我们花点时间仔细讨论一下，因为它们是本书反复出现的背景主题。

封装一词涵盖了两个密切相关的想法：简化行为和隐藏数据。在本次讨论中，我们使用第一种含义。我们通过识别代码中需要完成的任务并将该任务交给定义明确的对象或函数来封装行为。我们将该对象或函数称为 抽象。

看一下以下两段 Python 代码：

```py title='用urlib进行搜索'
import json
from urllib.request import urlopen
from urllib.parse import urlencode

params = dict(q='Sausages', format='json')
handle = urlopen('http://api.duckduckgo.com' + '?' + urlencode(params))
raw_text = handle.read().decode('utf8')
parsed = json.loads(raw_text)

results = parsed['RelatedTopics']
for r in results:
    if 'Text' in r:
        print(r['FirstURL'] + ' - ' + r['Text'])
```

```py title='用requests进行搜索'
import requests

params = dict(q='Sausages', format='json')
parsed = requests.get('http://api.duckduckgo.com/', params=params).json()

results = parsed['RelatedTopics']
for r in results:
    if 'Text' in r:
        print(r['FirstURL'] + ' - ' + r['Text'])
```

这两个代码都做了同样的事情：它们将表单编码的值提交给 URL，以便使用搜索引擎 API。但第二个代码清单更易于阅读和理解，因为它在更高的抽象级别上运行。

我们可以更进一步，识别和命名我们希望代码为我们执行的任务，并使用更高级别的抽象使其明确：

```py title='使用 duckduckgo 客户端库进行搜索'
import duckduckpy
for r in duckduckpy.query('Sausages').related_topics:
    print(r.first_url, ' - ', r.text)
```

使用抽象来封装行为是一个强大的工具，可以使代码更具表现力、更易于测试、更易于维护。

!!! note
    在面向对象 (OO) 领域的文献中，这种方法的经典特征之一被称为 [责任驱动设计](https://www.wirfs-brock.com/Design.html)；它使用角色和职责而不是任务。主要观点是从行为的角度来思考代码，而不是从数据或算法的角度来思考

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px">
    <p align='center'><strong style='font-size:1.6em;color:#186d7a'>抽象和ABC</strong></p>
    <p>
    在 Java 或 C# 等传统 OO 语言中，您可能会使用抽象基类 (ABC) 或接口来定义抽象。在 Python 中，您可以（有时我们会这样做）使用 ABC，但您也可以放心地使用鸭子类型。
    </p>
    <p>
    抽象可以仅仅意味着“您正在使用的东西的公共 API”——例如，函数名称加上一些参数。
    </p>
</div>

这本书中的大多数模式都涉及到选择抽象，因此在每一章中您会看大量示例。此外，[3.抽象](./f.Coupling%20and%20Abstractions.md) 特别讨论了选择抽象的一些启发式方法。


## <font color='red'>分层</font>

封装和抽象可以帮助我们隐藏细节并保护数据的一致性，但我们还需要注意对象和函数之间的交互。当一个函数、模块或对象使用另一个函数、模块或对象时，我们说一个函数、模块或对象依赖于另一个函数、模块或对象。这些依赖关系形成了一种网络或图。

在一个巨大的泥球中，依赖关系失控了（正如在图1）中所看到的）。更改图中的一个节点变得困难，因为它可能影响系统的许多其他部分。分层架构是解决这个问题的一种方式。在分层架构中，我们将代码划分为离散的类别或角色，并引入有关哪些类别的代码可以相互调用的规则。

最常见的示例之一是 图2. 中三层架构：

<figure markdown='span'>
    ![apwp 0002](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0002.png){ align=center }
    <figcaption><font size=2>图2.分层架构</font></figcaption>
</figure>

分层架构可能是构建业务软件最常见的模式。在这个模型中，我们有用户界面组件，可以是网页、API 或命令行；这些用户界面组件与包含业务规则和工作流的业务逻辑层进行通信；最后，我们有一个负责存储和检索数据的数据库层。

在本书的其余部分，我们将遵循一个简单的原则，系统地彻底改变这个模型。


## <font color='red'>依赖倒置原则</font>

你可能已经熟悉依赖倒置原则（DIP），因为它是 SOLID 中的 D。

不幸的是，我们无法像封装那样使用三个小代码清单来说明 DIP。但是，整个[part1](./c.Part1.md)本质上是一个在整个应用程序中实现 DIP 的工作示例，因此您将获得大量具体示例。

与此同时，我们可以谈谈DIP的正式定义：

1. 高级模块不应该依赖于低级模块。两者都应该依赖于抽象。
2. 抽象不应该依赖于细节。相反，细节应该依赖于抽象

但这到底意味着什么呢？让我们一点一点来分析一下。

高级模块是您的组织真正关心的代码。也许您在一家制药公司工作，您的高级模块处理患者和试验。也许您在一家银行工作，您的高级模块管理交易和交易所。软件系统的高级模块是处理我们现实世界概念的函数、类和包。

相比之下，低级模块是您的组织不关心的代码。您的人力资源部门不太可能对文件系统或网络套接字感兴趣。您很少与财务团队讨论 SMTP、HTTP 或 AMQP。对于我们的非技术利益相关者来说，这些低级概念并不有趣或相关。他们只关心高级概念是否正常工作。如果工资单按时运行，您的企业不太可能关心这是 Kubernetes 上运行的 cron 作业还是临时函数。

依赖于并不一定意味着导入或调用，而是一个模块知道或需要另一个模块的更一般的想法。

我们已经提到过抽象：它们是封装行为的简化接口，就像我们的`duckduckgo`模块封装了搜索引擎的 API 一样。

!!! quote
    <p>”计算机科学中的所有问题都可以通过增加一个间接层来解决。“</p>
    <p align='right'>— David Wheeler</p>

所以DIP的第一部分说我们的业务代码不应该依赖于技术细节；相反，两者都应该使用抽象。

为什么？从广义上讲，因为我们希望能够独立地更改它们。高级模块应该易于根据业务需求进行更改。在实践中，低级模块（细节）通常更难更改：考虑重构以更改函数名称，而不是定义、测试和部署数据库迁移来更改列名。我们不希望业务逻辑更改减慢速度，因为它们与低级基础架构细节紧密相关。但是，同样，能够在需要时更改基础架构细节（例如，考虑分片数据库）而无需更改业务层也很重要。在它们之间添加一个抽象（著名的额外间接层）允许两者（更）独立地进行更改。

第二部分更加神秘。“抽象不应该依赖于细节”似乎足够清晰，但“细节应该依赖于抽象”却难以想象。我们如何才能拥有不依赖于其抽象细节的抽象？到 [4.服务层](./g.Flask%20API%20and%20Service%20Layer.md) 时，将会有一个具体的例子，这应该会让这一切更清楚一些。


## <font color='red'>所有业务逻辑的存放地：领域模型</font>

但在彻底了解我们的三层架构之前，我们需要更多地讨论中间层：高级模块或业务逻辑。我们的设计出错的最常见原因之一是业务逻辑遍布应用程序的各个层，难以识别、理解和更改。

[1.领域模型](./d.Domain%20Modeling.md) 展示了如何使用领域模型模式构建业务层。[第一部分](./c.Part1.md) 中的其余模式展示了我们如何通过选择正确的抽象并持续应用 DIP ，使领域模型易于更改并不受低级关注的影响。
