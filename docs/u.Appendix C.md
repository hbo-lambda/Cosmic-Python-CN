# 附录C：更换基础设施：用csv完成所有工作


本附录旨在简要说明存储库、工作单元和服务层模式的优点。它旨在遵循[6.工作单元模式](./i.Unit%20of%20Work%20Pattern.md)。

就在我们完成 Flask API 的构建并准备发布时，企业向我们道歉，说他们还没有准备好使用我们的 API，并询问我们是否可以构建一个东西，从几个 CSV 中读取批次和订单，并输出带有分配的第三个 CSV。

通常，这种情况可能会让团队咒骂、吐口水，并在回忆录中做笔记。但我们不会！哦不，我们已经确保我们的基础设施问题与我们的领域模型和服务层很好地分离。切换到 CSV 将是一个简单的问题，只需编写几个新的`Repository`和`UnitOfWork`类，然后我们就可以重用领域层和服务层中的所有逻辑。

以下是一个 E2E 测试，向您展示 CSV 如何流入和流出：

```py title='第一个 CSV 测试 (tests/e2e/test_csv.py）'
def test_cli_app_reads_csvs_with_batches_and_orders_and_outputs_allocations(make_csv):
    sku1, sku2 = random_ref("s1"), random_ref("s2")
    batch1, batch2, batch3 = random_ref("b1"), random_ref("b2"), random_ref("b3")
    order_ref = random_ref("o")
    make_csv("batches.csv", [
        ["ref", "sku", "qty", "eta"],
        [batch1, sku1, 100, ""],
        [batch2, sku2, 100, "2011-01-01"],
        [batch3, sku2, 100, "2011-01-02"],
    ])
    orders_csv = make_csv("orders.csv", [
        ["orderid", "sku", "qty"],
        [order_ref, sku1, 3],
        [order_ref, sku2, 12],
    ])

    run_cli_script(orders_csv.parent)

    expected_output_csv = orders_csv.parent / "allocations.csv"
    with open(expected_output_csv) as f:
        rows = list(csv.reader(f))
    assert rows == [
        ["orderid", "sku", "qty", "batchref"],
        [order_ref, sku1, "3", batch1],
        [order_ref, sku2, "12", batch2],
    ]
```

深入研究并实施而不考虑存储库和所有那些爵士乐，您可以从以下类似内容开始：

```py title='我们的 CSV 读写器的初版 (src/bin/allocate-from-csv）'
#!/usr/bin/env python
import csv
import sys
from datetime import datetime
from pathlib import Path

from allocation.domain import model


def load_batches(batches_path):
    batches = []
    with batches_path.open() as inf:
        reader = csv.DictReader(inf)
        for row in reader:
            if row["eta"]:
                eta = datetime.strptime(row["eta"], "%Y-%m-%d").date()
            else:
                eta = None
            batches.append(
                model.Batch(
                    ref=row["ref"], sku=row["sku"], qty=int(row["qty"]), eta=eta
                )
            )
    return batches


def main(folder):
    batches_path = Path(folder) / "batches.csv"
    orders_path = Path(folder) / "orders.csv"
    allocations_path = Path(folder) / "allocations.csv"

    batches = load_batches(batches_path)

    with orders_path.open() as inf, allocations_path.open("w") as outf:
        reader = csv.DictReader(inf)
        writer = csv.writer(outf)
        writer.writerow(["orderid", "sku", "batchref"])
        for row in reader:
            orderid, sku = row["orderid"], row["sku"]
            qty = int(row["qty"])
            line = model.OrderLine(orderid, sku, qty)
            batchref = model.allocate(line, batches)
            writer.writerow([line.orderid, line.sku, batchref])


if __name__ == "__main__":
    main(sys.argv[1])
```

看起来还不错！我们正在重用我们的领域模型对象和领域服务。

但这是行不通的。现有的分配也需要成为我们永久 CSV 存储的一部分。我们可以编写第二个测试来强制我们改进：

```py title='另一个，具有现有分配（tests/e2e/test_csv.py）'
def test_cli_app_also_reads_existing_allocations_and_can_append_to_them(make_csv):
    sku = random_ref("s")
    batch1, batch2 = random_ref("b1"), random_ref("b2")
    old_order, new_order = random_ref("o1"), random_ref("o2")
    make_csv("batches.csv", [
        ["ref", "sku", "qty", "eta"],
        [batch1, sku, 10, "2011-01-01"],
        [batch2, sku, 10, "2011-01-02"],
    ])
    make_csv("allocations.csv", [
        ["orderid", "sku", "qty", "batchref"],
        [old_order, sku, 10, batch1],
    ])
    orders_csv = make_csv("orders.csv", [
        ["orderid", "sku", "qty"],
        [new_order, sku, 7],
    ])

    run_cli_script(orders_csv.parent)

    expected_output_csv = orders_csv.parent / "allocations.csv"
    with open(expected_output_csv) as f:
        rows = list(csv.reader(f))
    assert rows == [
        ["orderid", "sku", "qty", "batchref"],
        [old_order, sku, "10", batch1],
        [new_order, sku, "7", batch2],
    ]
```

我们可以继续修改并向该`load_batches`函数添加额外的代码行，以及某种跟踪和保存新分配的方法 - 但我们已经有一个模型可以做到这一点！ 它被称为我们的存储库和工作单元模式。

我们所需要做的（“我们所需要做的”）就是重新实现相同的抽象，但底层是 CSV，而不是数据库。正如您所看到的，这确实相对简单。


## <font color='red'>为 CSV 实现存储库和工作单元</font>

基于 CSV 的存储库可能如下所示。它抽象了从磁盘读取 CSV 的所有逻辑，包括必须读取两个不同的 CSV（一个用于批处理，一个用于分配）的事实，并且它只为我们提供了熟悉的`.list()`API，这提供了内存中域对象集合的幻觉：

```py title='使用 CSV 作为存储机制的存储库（src/allocation/service_layer/csv_uow.py）'
class CsvRepository(repository.AbstractRepository):
    def __init__(self, folder):
        self._batches_path = Path(folder) / "batches.csv"
        self._allocations_path = Path(folder) / "allocations.csv"
        self._batches = {}  # type: Dict[str, model.Batch]
        self._load()

    def get(self, reference):
        return self._batches.get(reference)

    def add(self, batch):
        self._batches[batch.reference] = batch

    def _load(self):
        with self._batches_path.open() as f:
            reader = csv.DictReader(f)
            for row in reader:
                ref, sku = row["ref"], row["sku"]
                qty = int(row["qty"])
                if row["eta"]:
                    eta = datetime.strptime(row["eta"], "%Y-%m-%d").date()
                else:
                    eta = None
                self._batches[ref] = model.Batch(ref=ref, sku=sku, qty=qty, eta=eta)
        if self._allocations_path.exists() is False:
            return
        with self._allocations_path.open() as f:
            reader = csv.DictReader(f)
            for row in reader:
                batchref, orderid, sku = row["batchref"], row["orderid"], row["sku"]
                qty = int(row["qty"])
                line = model.OrderLine(orderid, sku, qty)
                batch = self._batches[batchref]
                batch._allocations.add(line)

    def list(self):
        return list(self._batches.values())
```

CSV 的 UoW 如下所示：

```py title='CSV 的 UoW：commit = csv.writer (src/allocation/service_layer/csv_uow.py）'
class CsvUnitOfWork(unit_of_work.AbstractUnitOfWork):
    def __init__(self, folder):
        self.batches = CsvRepository(folder)

    def commit(self):
        with self.batches._allocations_path.open("w") as f:
            writer = csv.writer(f)
            writer.writerow(["orderid", "sku", "qty", "batchref"])
            for batch in self.batches.list():
                for line in batch._allocations:
                    writer.writerow(
                        [line.orderid, line.sku, line.qty, batch.reference]
                    )

    def rollback(self):
        pass
```

一旦我们有了这些，用于读取和写入批次和分配到 CSV 的 CLI 应用程序就会精简为它应该的样子——一些用于读取订单行的代码，以及一些调用我们 现有服务层的代码：

```py title='使用九行 CSV 进行分配 (src/bin/allocate-from-csv）'
def main(folder):
    orders_path = Path(folder) / "orders.csv"
    uow = csv_uow.CsvUnitOfWork(folder)
    with orders_path.open() as f:
        reader = csv.DictReader(f)
        for row in reader:
            orderid, sku = row["orderid"], row["sku"]
            qty = int(row["qty"])
            services.allocate(orderid, sku, qty, uow)
```

太棒了！现在你们都印象深刻了吗？

非常喜欢，

Bob和Harry