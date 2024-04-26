# 1.领域建模


本章将介绍如何以与 TDD 高度兼容的方式使用代码对业务流程进行建模。我们将讨论域建模的重要性，并介绍域建模的几个关键模式：实体、值对象和域服务。

图1 是我们领域模型模式的一个简单视觉占位符。我们将在本章中补充一些细节，随着我们进入其他章节，我们将围绕领域模型构建事物，但您应该始终能够在核心处找到这些小形状。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0101.png){ align=center }
    <figcaption><font size=2>图1.领域模型的占位符</font></figcaption>
</figure>


## <font color='red'>什么是领域模型</font>

在[介绍](./b.Introduction.md)中，我们使用术语业务逻辑层 来描述三层架构的中心层。在本书的其余部分，我们将改用术语领域模型。这是来自 DDD 社区的一个术语，可以更好地表达我们的意图。

领域是一个比较高级的说法，它指的是你试图解决的问题。作者目前在一家家具在线零售商工作。根据你所讨论的系统，领域可能涉及购买和采购、或者产品设计、或者物流和交付。大多数程序员的工作日都在尝试改进或使业务流程自动化；领域就是这些流程支持的一组活动。

模型是对捕捉有用属性的过程（或现象）的映射。人类非常擅长在头脑中建立事物的模型。例如，当有人向你扔球时，你几乎能无意识地预测它的运动，因为你对物体在空间中移动的方式有一个模型。当然，你的模型并不完美。人类对近光速或真空中物体行为的直觉很差，因为我们的模型从未被设计用于涵盖这些情况。这并不意味着模型是错误的，但确实意味着某些预测超出了它的范畴。

领域模型是企业主对其业务的思维导图。所有商务人士都有这样的思维导图——它们是人类思考复杂流程的方式。

你可以判断出他们何时会用这些思维导图，因为他们使用商业用语。在复杂系统上进行协作的人们之间自然会产生行话。

想象一下，你，我们不幸的读者，突然和你的朋友和家人乘坐外星飞船被送到距离地球数光年之外的地方，并且必须从基本原理出发，弄清楚如何导航回家。

在最初几天里，你可能只是随意按按钮，但很快你就会知道哪些按钮是干什么的，这样你就可以互相下达指令。你可能会说：“按下闪烁的小玩意旁边的红色按钮，然后把那个大杠杆扔到雷达装置旁边。”

几周之内，你就会更准确地使用词语来描述飞船的功能：“增加货舱三的氧气含量”或“打开小型推进器”。几个月后，你就会掌握描述整个复杂过程的语言：“开始着陆程序”或“准备跃迁”。这个过程会非常自然地发生，无需任何正式的努力来建立共享的词汇表。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>这不是一本DDD书籍，你应该读一本DDD书籍</strong></p>
<p>领域驱动设计（DDD）使领域建模的概念流行起来，通过专注于核心业务领域，在改变了人们设计软件的方面取得了巨大成功。我们在本书中介绍的许多架构模式，包括实体、聚合、值对象（见 7.聚合 ）和 存储库（在下一章中），都源自DDD传统。</p>
<p>简而言之，DDD 认为软件最重要的一点是它提供了问题的有用模型。如果我们正确建立了该模型，我们的软件就能创造价值并使新事物成为可能。</p>
<p>如果我们弄错了模型，它就会成为需要解决的障碍。在这本书中，我们可以展示构建领域模型的基础知识，并围绕它构建一个架构，使模型尽可能不受外部约束，以便于发展和改变。</p>
<p>但是，DDD 以及开发领域模型的流程、工具和技术还有很多内容。我们希望让您体验一下，并且极力鼓励您继续阅读一本合适的 DDD 书籍：</p>
<ul>
    <li>最初的“蓝皮书”，领域驱动设计，作者：Eric Evans（Addison-Wesley Professional）</li>
    <li>“红皮书”，《实施领域驱动设计》， 作者：Vaughn Vernon（Addison-Wesley Professional）</li>
</ul>
</div>

在现实的商业世界中也是如此。业务利益相关者使用的术语代表了对领域模型的精炼理解，其中复杂的思想和流程被浓缩为一个单词或短语。

当我们听到业务利益相关者使用陌生的词汇，或以特定方式使用术语时，我们应该倾听以理解更深层次的含义，并将他们来之不易的经验编码到我们的软件中。

在本书中，我们将使用真实领域的模型，具体来说是我们当前工作中的模型。MADE.com是一家成功的家具零售商。我们从世界各地的制造商那里采购家具，并销往欧洲各地。

当您购买沙发或咖啡桌时，我们必须想出最好的办法，将您的商品从波兰、中国或越南运送到您的客厅。

从高层次来看，我们有独立的系统负责采购库存、向客户销售库存以及向客户发货。中间系统需要通过根据客户的订单分配库存来协调该流程；请参见图2

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0102.png){ align=center }
    <figcaption><font size=2>图2.分配服务的环境图</font></figcaption>
</figure>

就本书而言，我们假设企业决定实施一种令人兴奋的新库存分配方式。到目前为止，企业一直根据仓库中的实际可用库存来显示库存和交货时间。如果仓库中的库存用完，产品将被列为“缺货”，直到制造商下一批货物到达。

创新点在于：如果我们有一个系统可以跟踪所有货物以及它们何时到达，我们就可以把这些船上的货物视为真正的库存，并视为我们库存的一部分，只是交货时间会稍微长一些。缺货的货物会越来越少，我们的销量也会越来越高，而且企业可以通过在国内仓库中保持较低的库存来节省成本。

但分配订单不再只是仓库系统中减少单个数量的简单事情。我们需要一个更复杂的分配机制。是时候进行一些领域建模了。


## <font color='red'>探索领域语言</font>

理解领域模型需要时间、耐心和便利贴。我们与业务专家进行了初步对话，并就领域模型的第一个最小版本的词汇表和一些规则达成一致。只要有可能，我们就会要求提供具体示例来说明每条规则。

我们确保用业务术语（ DDD 术语中的通用语言）来表达这些规则。我们为对象选择容易记住的标识符，以便更容易讨论示例。

以下展示了我们在与领域专家讨论分配时可能记录的一些笔记。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px">
    <p align='center'><strong style='font-size:1.6em;color:#186d7a'>分配注意事项</strong></p>
    <p>产品由SKU（发音为“skew”）标识，SKU是库存单位的缩写。客户下订单。订单由订单参考标识，由多个订单行组成，每行都有一个SKU和数量。例如：</p>
    <ul>
        <li>10个RED-CHAIR</li>
        <li>1个TASTELESS-LAMP</li>
    </ul>
    <p>采购部门订购小批量的库存。一批库存有唯一的ID（成为参考）、一个SKU和一个数量。</p>
    <p>我们需要将订单行分配 给批次。当我们将订单行分配给批次时，我们会将该特定批次的库存发送到客户的送货地址。当我们将x单位的库存分配给批次时，可用数量会减少x。例如：</p>
    <ul>
        <li>我们有一批 20 个 SMALL-TABLE，并且为 2 个 SMALL-TABLE 分配一个订单行。</li>
        <li>该批次应剩余 18 个 SMALL-TABLE。</li>
    </ul>
    <p>如果可用数量小于订单行的数量，则无法分配给批次。例如：</p>
    <ul>
        <li>我们有一个BLUE-CUSHION 数量为1的批次，，然后有一个2个BLUE-CUSHION的订单行。</li>
        <li>我们不应该将订单行分配给该批次。</li>
    </ul>
    <p>我们不能分配同一个订单行两次。例如：</p>
    <ul>
        <li>我们有一个 数量为10 BLUE-VASE的批次，然后我们将2个BLUE-VASE的订单行分配给它。</li>
        <li>如果我们再次将订单行分配给同一批次，该批次可用数量仍然为8。</li>
    </ul>
    <p>如果批次目前正在发货，或者它们可能在仓库库存中，则它们会有一个预计到达时间 (ETA)。我们优先分配给仓库库存，而不是发货批次。我们按照预计到达时间最早的顺序分配给发货批次。</p>
</div>


## <font color='red'>单元测试领域模型</font>

我们不会在本书中向您展示 TDD 的工作原理，但我们想向您展示如何根据此次业务对话构建模型。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px">
    <p align='center'><strong style='font-size:1.6em;color:#186d7a'>读者练习</strong></p>
    <p>为什么不试试自己解决这个问题呢？编写一些单元测试，看看能否以清晰、干净的代码捕捉到这些业务规则的本质（最好不查看我们下面提供的解决方案！）</p>
    <p>你会在<a src='https://github.com/cosmicpython/code/tree/chapter_01_domain_model_exercise'>GitHub</a>上找到一些占位符单元测试，但你可以从头开始，或者根据你的喜好组合/重写它们。</p>
</div>

我们的首次测试可能如下所示：

```py title='第一次分配测试（test_batch.py）'
def test_allocating_to_a_batch_reduces_the_available_quantity():
    batch = Batch("batch-001", "SMALL-TABLE", qty=20, eta=date.today())
    line = OrderLine("order-ref", "SMALL-TABLE", 2)

    batch.allocate(line)

    assert batch.available_quantity == 18
```

我们的单元测试的名称描述了我们希望从系统中看到的行为，我们使用的类和变量的名称取自业务术语。我们可以将这段代码展示给我们的非技术同事，他们会同意这正确地描述了系统的行为。

这是一个满足我们要求的领域模型：

```py title='批次领域模型的初版（model.py）'
@dataclass(frozen=True)  #(1) (2)
class OrderLine:
    orderid: str
    sku: str
    qty: int


class Batch:
    def __init__(self, ref: str, sku: str, qty: int, eta: Optional[date]):  #(2)
        self.reference = ref
        self.sku = sku
        self.eta = eta
        self.available_quantity = qty

    def allocate(self, line: OrderLine):  #(3)
        self.available_quantity -= line.qty
```

1. `OrderLine`是一个没有具体的行为的不可变的数据类。
2. 为了保持代码整洁，我们不会将导入显示在大多数代码中。我们希望您能猜到这是通过`from dataclasses import dataclass`，`typing.Optional`和`datetime.date`导入的。如果您想要仔细检查任何内容，您可以查看分支中每个章节的完整代码。([1.领域模型](https://github.com/cosmicpython/code/tree/chapter_01_domain_model))
3. 类型提示在 Python 中仍是一个有争议的问题。对于领域模型，它们有时可以帮助澄清或记录预期的参数是什么，而使用 IDE 的人往往对它们心存感激。您可能会认为在可读性方面付出的代价太高了

我们在这里的实现很简单：`Batch`只是包装了一个整数`available_quantity`，并在分配时减少该值。虽然我们编写了相当多的代码来执行从一个数减去另一个数的操作，但我们认为精确地建模会得到回报。

让我们编写一些新的失败测试：

```py title='测试我们可以分配的逻辑（test_batches.py）'
def make_batch_and_line(sku, batch_qty, line_qty):
    return (
        Batch("batch-001", sku, batch_qty, eta=date.today()),
        OrderLine("order-123", sku, line_qty),
    )

def test_can_allocate_if_available_greater_than_required():
    large_batch, small_line = make_batch_and_line("ELEGANT-LAMP", 20, 2)
    assert large_batch.can_allocate(small_line)

def test_cannot_allocate_if_available_smaller_than_required():
    small_batch, large_line = make_batch_and_line("ELEGANT-LAMP", 2, 20)
    assert small_batch.can_allocate(large_line) is False

def test_can_allocate_if_available_equal_to_required():
    batch, line = make_batch_and_line("ELEGANT-LAMP", 2, 2)
    assert batch.can_allocate(line)

def test_cannot_allocate_if_skus_do_not_match():
    batch = Batch("batch-001", "UNCOMFORTABLE-CHAIR", 100, eta=None)
    different_sku_line = OrderLine("order-123", "EXPENSIVE-TOASTER", 10)
    assert batch.can_allocate(different_sku_line) is False
```

这里没有什么太出乎意料的事情。我们重构了测试套件，这样我们就不会重复相同的代码行来为同一个 SKU 创建`Batch`和`OrderLine`；并且我们为新方法编写了四个简单的测试`can_allocate`。再次注意，我们使用的名称反映了我们领域专家的语言，我们商定的示例直接写入代码中。
我们也可以通过编写`Batch`的`can_allocate` 方法来直接实现这一点：

```py title='模型中的新方法（model.py）'
def can_allocate(self, line: OrderLine) -> bool:
    return self.sku == line.sku and self.available_quantity >= line.qty
```

到目前为止，我们可以通过简单地增加和减少`Batch.available_quantity`来管理实现，但当我们开始进行`deallocate()`测试时，我们将不得不采取更智能的解决方案：

```py title='这个测试将需要一个更智能的模型（test_batches.py）'
def test_can_only_deallocate_allocated_lines():
    batch, unallocated_line = make_batch_and_line("DECORATIVE-TRINKET", 20, 2)
    batch.deallocate(unallocated_line)
    assert batch.available_quantity == 20
```

在此测试中，我们断言，除非`Batch`先前分配了该`Orderline`，否则从该`Batch`中取消分配`OrderLine`不会产生任何影响。为了实现这一点，我们`Batch`需要了解哪些`OrderLine`已被分配。让我们看看实现：


```py title='领域模型现在跟踪分配（model.py）'
class Batch:
    def __init__(self, ref: str, sku: str, qty: int, eta: Optional[date]):
        self.reference = ref
        self.sku = sku
        self.eta = eta
        self._purchased_quantity = qty
        self._allocations = set()  # type: Set[OrderLine]

    def allocate(self, line: OrderLine):
        if self.can_allocate(line):
            self._allocations.add(line)

    def deallocate(self, line: OrderLine):
        if line in self._allocations:
            self._allocations.remove(line)

    @property
    def allocated_quantity(self) -> int:
        return sum(line.qty for line in self._allocations)

    @property
    def available_quantity(self) -> int:
        return self._purchased_quantity - self.allocated_quantity

    def can_allocate(self, line: OrderLine) -> bool:
        return self.sku == line.sku and self.available_quantity >= line.qty
```

UML中的模型

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0103.png){ align=center }
    <figcaption><font size=2>图3.UML中的模型</font></figcaption>
</figure>

现在有些进展了，批次（`Batch`）现在跟踪一组已分配的订单行（`OrderLine`）。当我们分配时，如果我们有足够的可用数量，我们只需将其添加到集合中。我们的`available_quantity`现在是一个计算的属性：已购买数量减去已分配数量。

是的，我们还可以做很多事情。让人有点不安的是`allocate()`和`deallocate()`可能会默默失败，但我们已经掌握了基础知识。

顺便说一下，使用`set`来处理`._allocations`，可以使我们能够轻松处理最后的测试，因为`set`中的项是唯一的：

```py title='最后一次批次测试!(test_batches.py）'
def test_allocation_is_idempotent():
    batch, line = make_batch_and_line("ANGULAR-DESK", 20, 2)
    batch.allocate(line)
    batch.allocate(line)
    assert batch.available_quantity == 18
```

目前，说领域模型太过简单，不值得用 DDD（甚至面向对象！）来思考，这可能是一个合理的批评。在现实生活中，会出现许多业务规则和边缘情况：客户可以要求在特定的未来日期交货，这意味着我们可能不想将它们分配给最早的批次。有些 SKU 不是分批的，而是直接从供应商处按需订购的，因此它们的逻辑不同。根据客户的位置，我们可以只分配给他们所在地区的部分仓库和货物 — 除了一些 SKU，如果我们所在地区缺货，我们很乐意从不同地区的仓库发货。等等。现实世界中的真实企业知道如何以比我们在页面上显示的速度更快地堆积复杂性！

但是，我们将把这个简单的领域模型作为更复杂事物的占位符，在本书的其余部分中扩展我们的简单领域模型，并将其插入到 API、数据库和电子表格的现实世界中。我们将看到，严格遵守封装和谨慎分层的原则将如何帮助我们避免陷入困境。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px">
    <p align='center'><strong style='font-size:1.6em;color:#186d7a'>更多类型，更多的类型提示</strong></p>
    如果您真的想要深入使用类型提示，您甚至可以使用<code>typing.NewType</code>来包装原始类型：
<pre>
    <code style='border: 1px solid;'>
from dataclasses import dataclass
from typing import NewType

Quantity = NewType("Quantity", int)
Sku = NewType("Sku", str)
Reference = NewType("Reference", str)
...

class Batch:
    def __init__(self, ref: Reference, sku: Sku, qty: Quantity):
        self.sku = sku
        self.reference = ref
        self._purchased_quantity = qty
    </code>
</pre>
<p>这将允许我们的类型检查器确保我们正确传递了参数。
<p>你认为这是好事还是坏事，还有待商榷。</p>
</div>


### <font color='red'>dataclasses非常适合值对象</font>

在之前的代码中，我们大量使用了"line"这个术语，但是什么是"line"（行）? 在我们的业务语言中，一个订单有多个订单项（line items），每个订单项都包含一个SKU和一个数量。我们可以想象，包含订单信息的简单YAML文件可能如下所示：

```yaml title='订单信息yaml'
Order_reference: 12345
Lines:
  - sku: RED-CHAIR
    qty: 25
  - sku: BLU-CHAIR
    qty: 25
  - sku: GRN-CHAIR
    qty: 25
```

请注意，虽然订单有一个可以唯一标识它的引用，但行（`line`）没有。即使我们将引用添加到`OrderLine`类中，它也不是唯一标识订单行本身的东西。

每当我们有一个具有数据但没有标识的业务概念时，通常我们选择使用值对象（*Value Object* ）来表示它。值对象是由其持有的数据进行唯一标识的领域对象；通常我们使它们不可变：

```py title='OrderLine是一个值对象'
@dataclass(frozen=True)
class OrderLine:
    orderid: OrderReference
    sku: ProductReference
    qty: Quantity
```

`dataclasse`或`namedtuples`给我们带来的一个好处是值相等性，也就是说，“具有相同orderid、sku和qty的两个订单行（`OrderLine`）是相等的”。

```py title='更多值对象示例'
from dataclasses import dataclass
from typing import NamedTuple
from collections import namedtuple

@dataclass(frozen=True)
class Name:
    first_name: str
    surname: str

class Money(NamedTuple):
    currency: str
    value: int

Line = namedtuple('Line', ['sku', 'qty'])

def test_equality():
    assert Money('gbp', 10) == Money('gbp', 10)
    assert Name('Harry', 'Percival') != Name('Bob', 'Gregory')
    assert Line('RED-CHAIR', 5) == Line('RED-CHAIR', 5)
```

这些值对象符合我们关于其值在现实中如何运作的直觉。我们讨论的是哪张10英镑纸币并不重要，因为它们都具有相同的价值。同样，如果名字和姓氏都匹配，两个名字是相等的；如果两个订单行具有想通的客户、sku和数量，则它们等同。但是，我们仍然可以对值对象添加复杂的行为。事实上，支持值的操作是很常见的；例如，数学运算：

```py title='值对象上的运算'
fiver = Money('gbp', 5)
tenner = Money('gbp', 10)

def can_add_money_values_for_the_same_currency():
    assert fiver + fiver == tenner

def can_subtract_money_values():
    assert tenner - fiver == fiver

def adding_different_currencies_fails():
    with pytest.raises(ValueError):
        Money('usd', 10) + Money('gbp', 10)

def can_multiply_money_by_a_number():
    assert fiver * 5 == Money('gbp', 25)

def multiplying_two_money_values_is_an_error():
    with pytest.raises(TypeError):
        tenner * fiver
```

为了让这些测试真正通过，你需要开始在我们的`Money`类上实现一些魔术方法：

```py title='在值对象实现运算'
@dataclass(frozen=True)
class Money:
    currency: str
    value: int

    def __add__(self, other) -> Money:
        if other.currency != self.currency:
            raise ValueError(f"Cannot add {self.currency} to {other.currency}")
        return Money(self.currency, self.value + other.value)
```


### <font color='red'>值对象和实体</font>

订单行（`OrderLine`）由其订单ID、SKU和数量唯一标识；如果我们更改其中一个值，我们现在有一个新的订单行（`OrderLine`）。这就是值对象的定义：任何仅由其数据标识并且没有长期身份的对象。但是批次（`Batch`）呢？批次是由一个引用（reference）标识的。

我们使用术语实体（`entity`）来描述具有长期身份的领域对象。在上一页，我们引入一个名字类（`Name`）作为值对象。如果我们取Harry Percival这个名字并更改其中的一个字母，我们会得到新的`Name`对象Barry Percival。

应该清楚的是Harry Percival不等于Barry Percival：

```py title='名字本身是无法改变的...'
def test_name_equality():
    assert Name("Harry", "Percival") != Name("Barry", "Percival")
```

但是Harry作为一个人又如何呢？人们确实会改变他们的名字、婚姻状况，甚至性别，但我们仍然会认出他们是同一个人。这是因为人类与名字不同，具有持久的身份：

```py title='但人可以...'
class Person:

    def __init__(self, name: Name):
        self.name = name


def test_barry_is_harry():
    harry = Person(Name("Harry", "Percival"))
    barry = harry

    barry.name = Name("Barry", "Percival")

    assert harry is barry and barry is harry
```

实体与值不同，具有身份相等性。我们可以更改它们的值，它们仍然可以被明确地识别为相同的东西。在我们的示例中，批次（`batches`）是实体。我们可以为批次分配订单行，或更改预计到货日期，它仍然是同一个实体。

通常，我们通过在实体上实现相等运算符来在代码中明确表示这一点：

```py title='实现相等运算符（model.py）'
class Batch:
    ...

    def __eq__(self, other):
        if not isinstance(other, Batch):
            return False
        return other.reference == self.reference

    def __hash__(self):
        return hash(self.reference)
```

Python的`__eq__`魔术方法定义了类在使用`==`运算符时的行为。

对于实体和值对象，还值得考虑`__hash__`方法的工作方式。这是Python用于控制对象在添加到集合或用作字典键时的行为的魔术方法；您可以在Python文档中找到更多信息。

对于值对象，哈希值应该基于所有值属性，并且我们应该确保对象是不可变的。通过在数据类上指定`@frozen=True`，我们可以实现这一点。

对于实体，最简单的选项是将哈希值设为None，这意味着对象是不可哈希的，例如不能在集合中使用。如果出于某种原因，您决定确实希望在实体上使用集合或字典操作，则哈希应基于属性，例如，该属性定义了实体随时间变化的唯一身份。您还应该尝试以某种方式将该`.reference`属性设为只读

!!! warning
    这是一个棘手的领域；在修改`__hash__`时，您也不应该忘记修改`__eq__`。如果您不确定自己在做什么，建议进一步阅读。我们的技术审阅员Hynek Schlawack编写的文章"[Python Hashes and Equality](https://hynek.me/articles/hashes-and-equality/)"是一个很好的起点。


## <font color='red'>并非所有东西都必须是对象：领域服务函数</font>

我们已经创建了一个表示批次的模型，但实际上我们需要做的是将订单行分配给一组特定的批次，这些批次代表了我们所有的库存。

!!! quote
    “有时，这根本就不是一回事。”
    <p align='right'>— Eric Evans</p>
    <p align='right'>Domain-Driven Design</p>

Evans讨论了在实体或值对象中没有自然归属的领域服务操作的概念。一个负责分配订单行的东西，给定一组批次，听起来很像一个函数，而且我们可以利用Python作为一种多范式语言的特性，直接将其实现为一个函数。

让我们看看如何通过测试驱动来创建这样一个函数：

```py title='测试我们的领域服务（test_allocate.py）'
def test_prefers_current_stock_batches_to_shipments():
    in_stock_batch = Batch("in-stock-batch", "RETRO-CLOCK", 100, eta=None)
    shipment_batch = Batch("shipment-batch", "RETRO-CLOCK", 100, eta=tomorrow)
    line = OrderLine("oref", "RETRO-CLOCK", 10)

    allocate(line, [in_stock_batch, shipment_batch])

    assert in_stock_batch.available_quantity == 90
    assert shipment_batch.available_quantity == 100


def test_prefers_earlier_batches():
    earliest = Batch("speedy-batch", "MINIMALIST-SPOON", 100, eta=today)
    medium = Batch("normal-batch", "MINIMALIST-SPOON", 100, eta=tomorrow)
    latest = Batch("slow-batch", "MINIMALIST-SPOON", 100, eta=later)
    line = OrderLine("order1", "MINIMALIST-SPOON", 10)

    allocate(line, [medium, earliest, latest])

    assert earliest.available_quantity == 90
    assert medium.available_quantity == 100
    assert latest.available_quantity == 100


def test_returns_allocated_batch_ref():
    in_stock_batch = Batch("in-stock-batch-ref", "HIGHBROW-POSTER", 100, eta=None)
    shipment_batch = Batch("shipment-batch-ref", "HIGHBROW-POSTER", 100, eta=tomorrow)
    line = OrderLine("oref", "HIGHBROW-POSTER", 10)
    allocation = allocate(line, [in_stock_batch, shipment_batch])
    assert allocation == in_stock_batch.reference
```

我们的服务可能如下所示：

```py title='我们的领域服务的独立函数（model.py）'
def allocate(line: OrderLine, batches: List[Batch]) -> str:
    batch = next(b for b in sorted(batches) if b.can_allocate(line))
    batch.allocate(line)
    return batch.reference
```


### <font color='red'>魔术方法让我们可以用Python中惯用的方式来使用模型</font>

您可能会喜欢或可能不会喜欢前面代码中使用`next()`，但我们非常确定您会同意能够在批次列表中使用`sorted()`是一种很好的、​​惯用的Python方式。

为了使其发挥作用，我们在我们的领域模型上实现了`__gt__`方法：

```py title='魔术方法可以表达领域语义（model.py）'
class Batch:
    ...

    def __gt__(self, other):
        if self.eta is None:
            return False
        if other.eta is None:
            return True
        return self.eta > other.eta
```

这很好。


### <font color='red'>异常也能表达领域概念</font>

我们还有最后一个概念要介绍：异常也可用于表达领域概念。在与领域专家的对话中，我们了解到由于缺货而无法分配订单的可能性，我们可以通过使用领域异常来捕获该情况：

```py title='测试缺货异常（test_allocate.py）'
def test_raises_out_of_stock_exception_if_cannot_allocate():
    batch = Batch("batch1", "SMALL-FORK", 10, eta=today)
    allocate(OrderLine("order1", "SMALL-FORK", 10), [batch])

    with pytest.raises(OutOfStock, match="SMALL-FORK"):
        allocate(OrderLine("order2", "SMALL-FORK", 1), [batch])
```

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>领域建模回顾</strong></p>
<p><strong>领域建模</strong></p>
<p>这是代码中最接近业务的部分，最有可能发生变化，也是您为业务提供最大价值的地方。使其易于理解和修改。</p>
<p><strong>区分实体和值对象</strong></p>
<p>值对象由其属性定义。通常最好将其实现为不可变类型。如果更改值对象的属性，则表示不同的对象。相反，实体具有随时间可能变化的属性，但仍然是同一个实体。定义唯一标识实体的内容很重要（通常是某种名称或引用字段）。</p>
<p><strong>并非所有东西都必须是对象</strong></p>
<p>Python是一种多范式语言，因此让代码中的“动词”成为函数。对于每个<code>FooManager</code>、<code>BarBuilder</code>或<code>BazFactory</code>，通常都可以使用更具表达力和可读性的<code>manage_foo()</code>、<code>build_bar()</code>或<code>get_baz()</code>函数。</p>
<p><strong>现在是应用最佳面向对象设计原则的时候了</strong></p>
<p>重新审视SOLID原则以及所有其他好的启发式方法，比如“有”与“是”，“优先使用组合而非继承”等等。</p>
<p><strong>你还需要考虑一致性边界和聚合</strong></p>
<p>但这是[第7章_聚合]的话题。</p>
</div>

我们不会让您对实现过程感到太过厌烦，但要注意的主要一点是，我们在通用语言中命名异常时要小心，就像我们命名实体、值对象和服务一样：

```py title='引发领域异常（model.py）'
class OutOfStock(Exception):
    pass


def allocate(line: OrderLine, batches: List[Batch]) -> str:
    try:
        batch = next(
        ...
    except StopIteration:
        raise OutOfStock(f"Out of stock for sku {line.sku}")
```

本章末尾的领域模型是我们最终成果的直观表示。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0104.png){ align=center }
    <figcaption><font size=2>图4.本章末尾的领域模型</font></figcaption>
</figure>


现在可能就够了！我们有一个领域服务，可用于我们的第一个用例。但首先我们需要一个数据库……
