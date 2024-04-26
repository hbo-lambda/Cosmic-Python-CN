# 附录B：模板项目结构


围绕[4.服务层](./g.Flask%20API%20and%20Service%20Layer.md)，我们从将所有内容放在一个文件夹中转变为更结构化的树，我们认为概述移动部件可能会很有趣。

!!! tip
    本附录的代码位于[GitHub](https://github.com/cosmicpython/code/tree/appendix_project_structure)上的appendix_project_structure 分支中：

    ```
    git clone https://github.com/cosmicpython/code.git
    cd code
    git checkout appendix_project_structure
    ```

基本文件夹结构如下：

``` yml title='项目树'
.
├── Dockerfile  (1)
├── Makefile  (2)
├── README.md
├── docker-compose.yml  (1)
├── license.txt
├── mypy.ini
├── requirements.txt
├── src  (3)
│   ├── allocation
│   │   ├── __init__.py
│   │   ├── adapters
│   │   │   ├── __init__.py
│   │   │   ├── orm.py
│   │   │   └── repository.py
│   │   ├── config.py
│   │   ├── domain
│   │   │   ├── __init__.py
│   │   │   └── model.py
│   │   ├── entrypoints
│   │   │   ├── __init__.py
│   │   │   └── flask_app.py
│   │   └── service_layer
│   │       ├── __init__.py
│   │       └── services.py
│   └── setup.py  (3)
└── tests  (4)
    ├── conftest.py  (4)
    ├── e2e
    │   └── test_api.py
    ├── integration
    │   ├── test_orm.py
    │   └── test_repository.py
    ├── pytest.ini  (4)
    └── unit
        ├── test_allocate.py
        ├── test_batches.py
        └── test_services.py
```

1. 我们的`docker-compose.yml`和`Dockerfile`是运行我们应用程序的容器的主要配置部分，它们也可以运行测试（用于 CI）。更复杂的项目可能会有多个 Dockerfile，尽管我们发现尽量减少镜像数量通常是一个好主意。
2. `Makefile`为开发人员（或 CI 服务器）在其正常工作流程中可能想要运行的所有典型命令提供了入口点：make build、make test等等。这是可选的。您可以直接使用`docker-compose`和`pytest`，但如果没有其他选择，最好将所有“常用命令”列在某个列表中，而且不像文档，`Makefile`是代码，因此它不太容易过时。
3. 我们应用程序的所有源代码（包括域模型、Flask 应用程序和基础架构代码）都位于 `src` 中的一个 Python 包中，我们使用 `pip install -e` 和 `setup.py` 文件进行安装。这使得导入变得容易。目前，此模块内的结构完全是扁平的，但对于更复杂的项目，您需要增加一个包含 `domain_model/`、`Infrastructure/`、`Services/`和`api/`的文件夹层次结构。
4. 测试位于其自己的文件夹中。子文件夹区分不同的测试类型并允许您单独运行它们。我们可以将共享fixtures（conftest.py ）保存在主测试文件夹中，并根据需要嵌套更多特定装置。这也是保存`pytest.ini`的地方。

!!! tip
    [pytest文档](https://docs.pytest.org/en/latest/explanation/goodpractices.html#choosing-a-test-layout-import-rules)在测试布局和可导入性方面确实很好。

让我们更详细地了解其中一些文件和概念。


## <font color='red'>容器内部和外部的环境变量、12 要素和配置</font>

我们在此尝试解决的基本问题是，我们需要针对以下内容进行不同的配置设置：

- 直接从您自己的开发机器运行代码或测试，也许与 Docker 容器的映射端口通信
- 在容器本身上运行，具有“真实”端口和主机名
- 不同的容器环境（开发、暂存、生产等）

[十二因素宣言](https://12factor.net/config)所建议的通过环境变量进行配置 将解决这个问题，但具体来说，我们如何在我们的代码和容器中实现它呢？


## <font color='red'>config.py</font>

每当我们的应用程序代码需要访问某些配置时，它都会从名为`config.py`的文件中获取它。以下是我们的应用程序中的几个示例：

```py title='示例配置函数（src/allocation/config.py）'
import os


def get_postgres_uri():  #(1)
    host = os.environ.get("DB_HOST", "localhost")  #(2)
    port = 54321 if host == "localhost" else 5432
    password = os.environ.get("DB_PASSWORD", "abc123")
    user, db_name = "allocation", "allocation"
    return f"postgresql://{user}:{password}@{host}:{port}/{db_name}"


def get_api_url():
    host = os.environ.get("API_HOST", "localhost")
    port = 5005 if host == "localhost" else 80
    return f"http://{host}:{port}"
```

1. 我们使用函数来获取当前配置，而不是导入时可用的常量，因为这允许客户端代码`os.environ`在需要时进行修改。
2. `config.py`还定义了一些默认设置，旨在从开发人员的本地机器运行代码时起作用。

如果您厌倦了手动编写基于环境的配置函数，那么一个名为[environ-config](https://github.com/hynek/environ-config)的优雅 Python 包 值得一看。

!!! tip
    不要让这个配置模块变成一个垃圾场，里面充满了与配置只有模糊关系的东西，然后到处导入。保持事物不变，只通过环境变量修改它们。如果您决定使用[引导脚本](./q.Dependency%20Injection.md)，您可以使其成为导入配置的唯一位置（测试除外）。


## <font color='red'>Docker-Compose 和容器配置</font>

我们使用轻量级的 Docker 容器编排工具`docker-compose`。它的主要配置是通过 YAML 文件进行的：[ 5 ]

```yml title='docker-compose 配置文件（docker-compose.yml）'
version: "3"
services:

  app:  #(1)
    build:
      context: .
      dockerfile: Dockerfile
    depends_on:
      - postgres
    environment:  #(3)
      - DB_HOST=postgres  (4)
      - DB_PASSWORD=abc123
      - API_HOST=app
      - PYTHONDONTWRITEBYTECODE=1  #(5)
    volumes:  #(6)
      - ./src:/src
      - ./tests:/tests
    ports:
      - "5005:80"  (7)


  postgres:
    image: postgres:9.6  #(2)
    environment:
      - POSTGRES_USER=allocation
      - POSTGRES_PASSWORD=abc123
    ports:
      - "54321:5432"
```

1. 在`docker-compose`文件中，我们定义了应用所需的不同服务 （容器）。通常一个主镜像包含我们所有的代码，我们可以使用它来运行我们的 API、测试或任何其他需要访问域模型的服务。
2. 您可能还有其他基础设施服务，包括数据库。在生产中，您可能不会为此使用容器；您可能有云提供商，但`docker-compose`为我们提供了一种为开发或 CI 生成类似服务的方法。
3. 该`environment节`允许您设置容器的环境变量、从 Docker 集群内部看到的主机名和端口。如果您有足够多的容器，以至于这些部分中的信息开始重复，您可以改用`environment_file`。我们通常称之为`container.env`。
4. 在集群内部，`docker-compose`设置网络，以便容器可以通过以其服务名称命名的主机名相互访问。
5. 生产提示：如果您正在挂载卷以在本地开发机器和容器之间共享源文件夹，`PYTHONDONTWRITEBYTECODE`环境变量会告诉 Python 不要写入`.pyc`文件，这样可以避免数百万个根拥有的文件散布在整个本地文件系统中，删除这些文件很烦人，而且还会导致奇怪的 Python 编译器错误。
6. 安装我们的源代码和测试代码意味着volumes我们不需要在每次更改代码时重建我们的容器。
7. 该ports部分允许我们将容器内部的端口暴露给外界——这些端口对应于我们在`config.py`中设置的默认端口。

!!! note
    在 Docker 内部，其他容器可通过以其服务名称命名的主机名来访问。在 Docker 外部，它们可在`localhost`上访问，端口为`ports`部分中定义的端口。


## <font color='red'>将源代码安装为软件包</font>

我们的所有应用程序代码（实际上除了测试之外的所有代码）都位于 src文件夹中：

```shell title='src 文件夹'
├── src
│   ├── allocation  #(1)
│   │   ├── config.py
│   │   └── ...
│   └── setup.py  (2)
```

1. 子文件夹定义顶级模块名称。如果您愿意，可以有多个。
2. `setup.py`是使其可通过 `pip` 安装所需的文件，如下所示。

```py title='pip 可在三行中安装模块（src/setup.py）'
from setuptools import setup

setup(
    name="allocation", version="0.1", packages=["allocation"],
)
```

这就是你所需要的。`packages=`指定要作为顶级模块安装的子文件夹的名称。该`name`条目只是表面的，但它是必需的。对于实际上永远不会进入 PyPI 的包，它就足够了。


## <font color='red'>Dockerfile</font>

Dockerfile 将会非常特定于项目，但是您将会看到以下几个关键阶段：

```dockerfile title='我们的Dockerfile（Dockerfile）'
FROM python:3.9-slim-buster

(1)
# RUN apt install gcc libpq (no longer needed bc we use psycopg2-binary)

(2)
COPY requirements.txt /tmp/
RUN pip install -r /tmp/requirements.txt

(3)
RUN mkdir -p /src
COPY src/ /src/
RUN pip install -e /src
COPY tests/ /tests/

(4)
WORKDIR /src
ENV FLASK_APP=allocation/entrypoints/flask_app.py FLASK_DEBUG=1 PYTHONUNBUFFERED=1
CMD flask run --host=0.0.0.0 --port=80
```

1. 安装系统级依赖项
2. 安装我们的 Python 依赖项（你可能希望将开发依赖项与生产依赖项分离；为简单起见，我们这里没有这样做）
3. 复制并安装我们的源代码
4. 可选择配置默认启动命令（您可能会从命令行多次覆盖该命令）

!!! tip
    需要注意的一点是，我们按照事物可能更改的频率顺序进行安装。这使我们能够最大限度地重用 Docker 构建缓存。我无法告诉您这堂课背后有多少痛苦和挫折。有关此提示以及更多 Python Dockerfile 改进提示，请查看 “[生产就绪 Docker 打包](https://pythonspeed.com/docker)”。


## <font color='red'>测试</font>

我们的测试与其他所有测试一起保存，如下所示：

```shell title='测试文件夹树'
└── tests
    ├── conftest.py
    ├── e2e
    │   └── test_api.py
    ├── integration
    │   ├── test_orm.py
    │   └── test_repository.py
    ├── pytest.ini
    └── unit
        ├── test_allocate.py
        ├── test_batches.py
        └── test_services.py
```

这里没有什么特别聪明的东西，只是将您可能想要单独运行的不同测试类型进行分离，以及一些用于常用装置、配置等的文件。

测试文件夹中没有`src`文件夹或`setup.py`，因为我们通常不需要使测试可通过 `pip` 安装，但如果您在导入路径方面遇到困难，您可能会发现它会有所帮助。


## <font color='red'>总结</font>

这些是我们的基本组成部分：

- 源代码位于src文件夹中，可使用`setup.py`通过 `pip` 安装
- 一些 Docker 配置用于启动尽可能镜像生产的本地集群
- 通过环境变量进行配置，集中在一个名为`config.py`的 Python 文件中，默认情况下允许在容器外部运行
- 一个用于有用的命令行命令的 Makefile

我们怀疑任何人最终都会得到与我们完全相同的解决方案，但我们希望您能在这里找到一些启发。