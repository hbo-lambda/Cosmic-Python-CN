# 附录D：Django 的存储库和工作单元模式


假设您想使用 Django 而不是 SQLAlchemy 和 Flask。情况会如何？第一件事是选择安装位置。我们将其放在主配置代码旁边的单独包中：

```shell
├── src
│   ├── allocation
│   │   ├── __init__.py
│   │   ├── adapters
│   │   │   ├── __init__.py
...
│   ├── djangoproject
│   │   ├── alloc
│   │   │   ├── __init__.py
│   │   │   ├── apps.py
│   │   │   ├── migrations
│   │   │   │   ├── 0001_initial.py
│   │   │   │   └── __init__.py
│   │   │   ├── models.py
│   │   │   └── views.py
│   │   ├── django_project
│   │   │   ├── __init__.py
│   │   │   ├── settings.py
│   │   │   ├── urls.py
│   │   │   └── wsgi.py
│   │   └── manage.py
│   └── setup.py
└── tests
    ├── conftest.py
    ├── e2e
    │   └── test_api.py
    ├── integration
    │   ├── test_repository.py
...
```

!!! tip
    本附录的代码位于[GitHub](https://github.com/cosmicpython/code/tree/appendix_django)上的appendix_django 分支中：

    ```
    git clone https://github.com/cosmicpython/code.git
    cd code
    git checkout appendix_django
    ```

    代码示例紧接着[6.工作单元模式](./i.Unit%20of%20Work%20Pattern.md)的结尾。


## <font color='red'>Django 的存储库模式</font>

我们使用了一个名为的插件`pytest-django`来帮助测试数据库管理。

重写第一个存储库测试是一个最小的变化 - 只需调用 Django ORM / QuerySet 语言重写一些原始 SQL：

```py title='第一个存储库测试已调整（tests/integration/test_repository.py）'
from djangoproject.alloc import models as django_models


@pytest.mark.django_db
def test_repository_can_save_a_batch():
    batch = model.Batch("batch1", "RUSTY-SOAPDISH", 100, eta=date(2011, 12, 25))

    repo = repository.DjangoRepository()
    repo.add(batch)

    [saved_batch] = django_models.Batch.objects.all()
    assert saved_batch.reference == batch.reference
    assert saved_batch.sku == batch.sku
    assert saved_batch.qty == batch._purchased_quantity
    assert saved_batch.eta == batch.eta
```

第二个测试稍微复杂一些，因为它有分配，但它仍然由熟悉的 Django 代码组成：

```py title='第二个存储库测试更加复杂（tests/integration/test_repository.py）'
@pytest.mark.django_db
def test_repository_can_retrieve_a_batch_with_allocations():
    sku = "PONY-STATUE"
    d_line = django_models.OrderLine.objects.create(orderid="order1", sku=sku, qty=12)
    d_batch1 = django_models.Batch.objects.create(
        reference="batch1", sku=sku, qty=100, eta=None
    )
    d_batch2 = django_models.Batch.objects.create(
        reference="batch2", sku=sku, qty=100, eta=None
    )
    django_models.Allocation.objects.create(line=d_line, batch=d_batch1)

    repo = repository.DjangoRepository()
    retrieved = repo.get("batch1")

    expected = model.Batch("batch1", sku, 100, eta=None)
    assert retrieved == expected  # Batch.__eq__ only compares reference
    assert retrieved.sku == expected.sku
    assert retrieved._purchased_quantity == expected._purchased_quantity
    assert retrieved._allocations == {
        model.OrderLine("order1", sku, 12),
    }
```

实际存储库的最终样子如下：

```py title='Django 存储库（src/allocation/adapters/repository.py）'
class DjangoRepository(AbstractRepository):
    def add(self, batch):
        super().add(batch)
        self.update(batch)

    def update(self, batch):
        django_models.Batch.update_from_domain(batch)

    def _get(self, reference):
        return (
            django_models.Batch.objects.filter(reference=reference)
            .first()
            .to_domain()
        )

    def list(self):
        return [b.to_domain() for b in django_models.Batch.objects.all()]
```

你可以看到，该实现依赖于 Django 模型，该模型具有一些自定义方法，用于与我们的域模型进行转换。


### <font color='red'>Django ORM 类上的自定义方法用于与域模型进行转换</font>

这些自定义方法看起来像这样：

```py title='Django ORM 具有用于域模型转换的自定义方法（src/djangoproject/alloc/models.py）'
from django.db import models
from allocation.domain import model as domain_model


class Batch(models.Model):
    reference = models.CharField(max_length=255)
    sku = models.CharField(max_length=255)
    qty = models.IntegerField()
    eta = models.DateField(blank=True, null=True)

    @staticmethod
    def update_from_domain(batch: domain_model.Batch):
        try:
            b = Batch.objects.get(reference=batch.reference)  #(1)
        except Batch.DoesNotExist:
            b = Batch(reference=batch.reference)  #(1)
        b.sku = batch.sku
        b.qty = batch._purchased_quantity
        b.eta = batch.eta  #(2)
        b.save()
        b.allocation_set.set(
            Allocation.from_domain(l, b)  #(3)
            for l in batch._allocations
        )

    def to_domain(self) -> domain_model.Batch:
        b = domain_model.Batch(
            ref=self.reference, sku=self.sku, qty=self.qty, eta=self.eta
        )
        b._allocations = set(
            a.line.to_domain()
            for a in self.allocation_set.all()
        )
        return b


class OrderLine(models.Model):
    #...
```

1. 对于值对象，`objects.get_or_create`可以工作，但对于实体，您可能需要显式的 `try-get/except` 来处理更新或插入。
2. 我们在这里展示了最复杂的示例。如果您决定这样做，请注意会有样板！幸运的是，这不是非常复杂的样板。
3. 关系也需要一些谨慎、定制的处理。

!!! note
    和[2.存储库模式](./e.Repository%20Pattern.md)​​一样，我们使用依赖反转。ORM（Django）依赖于模型，而不是相反。


## <font color='red'>Django 的工作单元模式</font>

测试不会发生太大变化：

```py title='改编的 UoW 测试（tests/integration/test_uow.py）'
def insert_batch(ref, sku, qty, eta):  #(1)
    django_models.Batch.objects.create(reference=ref, sku=sku, qty=qty, eta=eta)


def get_allocated_batch_ref(orderid, sku):  #(1)
    return django_models.Allocation.objects.get(
        line__orderid=orderid, line__sku=sku
    ).batch.reference


@pytest.mark.django_db(transaction=True)
def test_uow_can_retrieve_a_batch_and_allocate_to_it():
    insert_batch("batch1", "HIPSTER-WORKBENCH", 100, None)

    uow = unit_of_work.DjangoUnitOfWork()
    with uow:
        batch = uow.batches.get(reference="batch1")
        line = model.OrderLine("o1", "HIPSTER-WORKBENCH", 10)
        batch.allocate(line)
        uow.commit()

    batchref = get_allocated_batch_ref("o1", "HIPSTER-WORKBENCH")
    assert batchref == "batch1"


@pytest.mark.django_db(transaction=True)  #(2)
def test_rolls_back_uncommitted_work_by_default():
    ...

@pytest.mark.django_db(transaction=True)  #(2)
def test_rolls_back_on_error():
    ...
```

1. 因为我们在这些测试中几乎没有辅助函数，所以测试的实际主体与 SQLAlchemy 几乎相同。
2. 需要`pytest-django` `mark.django_db(transaction=True)`测试我们的自定义事务/回滚行为。

虽然我尝试了好几次才找到哪个 Django 事务魔法调用可以起作用，但实现起来相当简单：

```py title='适用于 Django 的 UoW（src/allocation/service_layer/unit_of_work.py）'
class DjangoUnitOfWork(AbstractUnitOfWork):
    def __enter__(self):
        self.batches = repository.DjangoRepository()
        transaction.set_autocommit(False)  #(1)
        return super().__enter__()

    def __exit__(self, *args):
        super().__exit__(*args)
        transaction.set_autocommit(True)

    def commit(self):
        for batch in self.batches.seen:  #(3)
            self.batches.update(batch)  #(3)
        transaction.commit()  #(2)

    def rollback(self):
        transaction.rollback()  #(2)
```

1. `set_autocommit(False)`是告诉 Django 停止立即自动提交每个 ORM 操作并开始事务的最佳方式。
2. 然后我们使用显式回滚和提交。
3. 一个困难：因为与 SQLAlchemy 不同，我们不会对域模型实例本身进行检测，所以命令`commit()`需要明确遍历每个存储库接触过的所有对象，并手动将它们更新回 ORM。


## <font color='red'>API：Django 视图是适配器</font>

Django views.py文件最终与旧的flask_app.py几乎相同，因为我们的架构意味着它是我们服务层的一个非常薄的包装器（顺便说一下，它根本没有改变）：

```py title='Flask 应用程序 → Django 视图 (src/djangoproject/alloc/views.py)'
os.environ["DJANGO_SETTINGS_MODULE"] = "djangoproject.django_project.settings"
django.setup()


@csrf_exempt
def add_batch(request):
    data = json.loads(request.body)
    eta = data["eta"]
    if eta is not None:
        eta = datetime.fromisoformat(eta).date()
    services.add_batch(
        data["ref"], data["sku"], data["qty"], eta,
        unit_of_work.DjangoUnitOfWork(),
    )
    return HttpResponse("OK", status=201)


@csrf_exempt
def allocate(request):
    data = json.loads(request.body)
    try:
        batchref = services.allocate(
            data["orderid"],
            data["sku"],
            data["qty"],
            unit_of_work.DjangoUnitOfWork(),
        )
    except (model.OutOfStock, services.InvalidSku) as e:
        return JsonResponse({"message": str(e)}, status=400)

    return JsonResponse({"batchref": batchref}, status=201)
```


## <font color='red'>为什么这一切这么难？</font>

好的，它起作用了，但感觉比 Flask/SQLAlchemy 更费力。为什么呢？

从底层来看，主要原因是 Django 的 ORM 工作方式不同。我们没有与 SQLAlchemy 经典映射器相当的映射器，因此我们的 `ActiveRecord` 和我们的域模型不能是同一个对象。相反，我们必须在存储库后面构建一个手动转换层。这需要做更多的工作（尽管一旦完成，持续的维护负担应该不会太高）。

因为 Django 与数据库紧密耦合，所以您必须`pytest-django`从第一行代码开始使用类似的帮助程序并仔细考虑测试数据库，而当我们开始使用纯域模型时则不必这样做。

但从更高层次来看，Django 如此出色的全部原因在于，它的设计初衷是让开发者能够轻松构建 CRUD 应用，同时尽量减少样板代码。但我们这本书的重点在于，当你的应用不再是简单的 CRUD 应用时，你该怎么做。

此时，Django 带来的阻碍比帮助更多。Django 管理员之类的东西，刚开始使用时非常棒，但如果应用程序的整个目的是围绕状态更改工作流构建一套复杂的规则和模型，那么它就会变得非常危险。Django 管理员可以绕过所有这些。


## <font color='red'>如果你已经用了 Django，该怎么办</font>

那么，如果你想将本书中的某些模式应用到 Django 应用中，你应该怎么做？我们的建议如下：

1. 存储库和工作单元模式需要做大量的工作。它们在短期内能为您带来的主要好处是更快的单元测试，因此请评估这种好处是否值得。从长远来看，它们会将您的应用与 Django 和数据库分离，因此如果您预计要从其中任何一个迁移，那么存储库和 UoW 是一个好主意。
2. 如果你在views.py中看到大量重复内容，那么服务层模式可能会引起你的兴趣。这是一种将用例与 Web 端点区分开来思考的好方法。
3. 从理论上讲，您仍然可以使用 Django 模型进行 DDD 和领域建模，尽管它们与数据库紧密耦合；迁移可能会减慢您的速度，但不会造成致命影响。因此，只要您的应用程序不太复杂且测试不太慢，您就可以从胖模型方法中获益：将尽可能多的逻辑推送到您的模型中，并应用实体、值对象和聚合等模式。但是，请注意以下警告。

话虽如此， Django 社区中传言称，人们发现胖模型方法本身存在可扩展性问题，特别是在管理应用程序之间的相互依赖性方面。在这种情况下，提取业务逻辑或领域层以位于视图和表单与models.py之间有很多好处，然后您可以将其保持在尽可能小的水平。


## <font color='red'>一路走来</font>

假设您正在开发一个 Django 项目，但您不确定该项目是否会变得足够复杂以保证我们推荐的模式，但您仍然希望采取一些措施来让您的生活更轻松，无论是在中期还是以后想要迁移到我们的某些模式。考虑以下几点：

1. 我们听到的一条建议是从第一天开始就将logic.py放入每个 Django 应用中。这为您提供了一个放置业务逻辑的地方，并使您的表单、视图和模型不受业务逻辑的影响。它可以成为以后转向完全解耦的域模型和/或服务层的垫脚石。
2. 业务逻辑层可能开始使用 Django 模型对象，后来才完全脱离框架并处理普通的 Python 数据结构。
3. 对于读取方面，您可以通过将读取集中到一个地方来获得 CQRS 的一些好处，避免 ORM 调用散布在各处。
4. 在分离读取模块和域逻辑模块时，可能值得将自己与 Django 应用层次结构分离。业务问题将贯穿其中。

!!! note
    我们想向 David Seddon 和 Ashia Zawaduk 致谢，感谢他们讨论了本附录中的一些想法。他们尽了最大努力阻止我们针对我们并没有足够亲身体验的话题说出任何愚蠢的话，但他们可能失败了。

有关处理现有应用程序的更多想法和实际生活经验，请参阅[结语](./r.Epilogue.md)。