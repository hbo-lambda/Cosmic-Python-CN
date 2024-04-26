# 3.简短插曲：耦合和抽象


亲爱的读者，请允许我们在抽象的主题上稍作偏离。我们已经讨论了很多关于抽象的内容。例如，存储库模式就是对永久存储的一种抽象。但是，什么才是一个好的抽象？我们从抽象中想要什么？它们与测试有何关联？

!!! tip
    本章的代码位于[GitHub](https://github.com/cosmicpython/code/tree/appendix_project_structure)上的 Chapter_03_abstractions 分支中：
    ```
    git clone https://github.com/cosmicpython/code.git
    git checkout chapter_03_abstractions
    ```

本书的一个关键主题隐藏在精美的模式中，那就是我们可以使用简单的抽象来隐藏杂乱的细节。当我们为了好玩或练习而编写代码时，我们可以自由发挥想法，敲定方案并积极重构。然而，在大型系统中，我们会受到系统中其他地方做出的决策的限制。

当我们因为担心组件 B 会损坏而无法更改组件 A 时，我们称组件已耦合。从局部来看，耦合是一件好事：这表明我们的代码可以协同工作，每个组件都支持其他组件，所有组件都像手表的齿轮一样配合到位。用行话来说，当耦合元素之间存在高内聚力时，我们称这种情况有效。

从总体上看，耦合是一件麻烦事：它增加了更改代码的风险和成本，有时甚至到了我们觉得根本无法进行任何更改的地步。这就是泥球模式的问题所在：随着应用程序的增长，如果我们无法防止没有内聚力的元素之间的耦合，那么这种耦合就会超线性增加，直到我们不再能够有效地更改我们的系统。

我们可以通过抽象细节（图2） 来降低系统内的耦合度（图1） 。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0301.png){ align=center }
    <figcaption><font size=2>图1.大量耦合</font></figcaption>
</figure>

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0302.png){ align=center }
    <figcaption><font size=2>图2.更少的耦合</font></figcaption>
</figure>

在两个图中，我们都有一对子系统，其中一个依赖于另一个。在图1中，两者之间的耦合度很高；箭头的数量表示两者之间存在多种依赖关系。如果我们需要更该系统B，那么更该很可能会波及到系统A。

然而，在图2中，我们通过插入一个新的、更简单的抽象来减少耦合度。由于更简单，系统A对抽象的依赖更少。这个抽象通过隐藏系统B的复杂细节来保护我们免受更改的影响——我们可以改变右边的箭头，而不改变左边的箭头。


## <font color='red'>抽象化状态有助于可测试性</font>

让我们看一个例子。假设我们要编写代码来同步两个文件目录，我们将其称为源和目标：

- 如果文件存在于源中但不存在于目标中，则复制该文件。
- 如果源中存在文件，但其名称与目标文件中的名称不同，则重命名目标文件以匹配。
- 如果文件存在于目标中但不存在于源中，则将其删除。

我们的第一个和第三个要求很简单：我们可以比较两个路径列表。不过，我们的第二个要求比较棘手。要检测重命名，我们必须检查文件的内容。为此，我们可以使用 MD5 或 SHA-1 等哈希函数。从文件生成 SHA-1 哈希的代码非常简单：

``` py title='对文件做哈希处理（sync.py）'
BLOCKSIZE = 65536


def hash_file(path):
    hasher = hashlib.sha1()
    with path.open("rb") as file:
        buf = file.read(BLOCKSIZE)
        while buf:
            hasher.update(buf)
            buf = file.read(BLOCKSIZE)
    return hasher.hexdigest()
```

现在我们需要编写一些决定做什么的部分——如果你愿意的话，可以称之为业务逻辑。

当我们必须从基本原理出发解决问题时，我们通常会尝试编写一个简单的实现，然后重构以获得更好的设计。我们将在整本书中使用这种方法，因为这是我们在现实世界中编写代码的方式：从问题最小部分的解决方案开始，然后迭代地使解决方案更丰富、设计更好。

我们的第一个hacking的方法大致如下：

``` py title='基础同步算法（sync.py）'
import hashlib
import os
import shutil
from pathlib import Path


def sync(source, dest):
    # Walk the source folder and build a dict of filenames and their hashes
    source_hashes = {}
    for folder, _, files in os.walk(source):
        for fn in files:
            source_hashes[hash_file(Path(folder) / fn)] = fn

    seen = set()  # Keep track of the files we've found in the target

    # Walk the target folder and get the filenames and hashes
    for folder, _, files in os.walk(dest):
        for fn in files:
            dest_path = Path(folder) / fn
            dest_hash = hash_file(dest_path)
            seen.add(dest_hash)

            # if there's a file in target that's not in source, delete it
            if dest_hash not in source_hashes:
                dest_path.remove()

            # if there's a file in target that has a different path in source,
            # move it to the correct path
            elif dest_hash in source_hashes and fn != source_hashes[dest_hash]:
                shutil.move(dest_path, Path(folder) / source_hashes[dest_hash])

    # for every file that appears in source but not target, copy the file to
    # the target
    for source_hash, fn in source_hashes.items():
        if source_hash not in seen:
            shutil.copy(Path(source) / fn, Path(dest) / fn)
```

太棒了！我们有一些代码，看起来还不错，但在我们将其运行在硬盘上之前，也许我们应该对其进行测试。那么，我们应该如何测试这种东西呢？

``` py title='一些端到端测试（test_sync.py）'
def test_when_a_file_exists_in_the_source_but_not_the_destination():
    try:
        source = tempfile.mkdtemp()
        dest = tempfile.mkdtemp()

        content = "I am a very useful file"
        (Path(source) / "my-file").write_text(content)

        sync(source, dest)

        expected_path = Path(dest) / "my-file"
        assert expected_path.exists()
        assert expected_path.read_text() == content

    finally:
        shutil.rmtree(source)
        shutil.rmtree(dest)


def test_when_a_file_has_been_renamed_in_the_source():
    try:
        source = tempfile.mkdtemp()
        dest = tempfile.mkdtemp()

        content = "I am a file that was renamed"
        source_path = Path(source) / "source-filename"
        old_dest_path = Path(dest) / "dest-filename"
        expected_dest_path = Path(dest) / "source-filename"
        source_path.write_text(content)
        old_dest_path.write_text(content)

        sync(source, dest)

        assert old_dest_path.exists() is False
        assert expected_dest_path.read_text() == content

    finally:
        shutil.rmtree(source)
        shutil.rmtree(dest)
```

哇哦，对于两个简单的案例来说，设置工作量太大了！问题在于，我们的领域逻辑“找出两个目录之间的差异”与I/O代码紧密相关。我们无法在不调用`pathlib`、`shutil`和`hashlib`模块的情况下运行我们的差异算法。

更麻烦的是，即使按照我们当前的需求，我们也没有编写足够的测试：当前的实现存在几个错误（例如，`shutil.move()`是错误的）。获得适当的覆盖范围并揭示这些错误意味着编写更多测试，但如果它们都像前面的测试一样难以处理，那么很快就会变得非常痛苦。

此外，我们的代码不太可扩展。想象试图实现一个`--dry-run`标志，让我们的代码只打印出它要做什么，而不是真正去做。或者如果我们想同步到远程服务器或云存储怎么办？

我们的高级代码与低级细节耦合在一起，这让事情变得很困难。随着我们考虑的场景变得越来越复杂，我们的测试将变得越来越难以处理。我们当然可以重构这些测试（例如，一些清理工作可以放在`pytest fixtures`中），但只要我们执行文件系统操作，它们就会变得缓慢，难以读写。


## <font color='red'>选择正确的抽象</font>

我们应该做些什么来重写我们的代码以使其更易于测试？

首先，我们需要考虑我们的代码需要从文件系统获取什么。通读代码，我们可以看到发生了三件不同的事情。我们可以将它们视为代码的三个不同职责：

- 我们通过使用`os.walk`来查询文件系统，并确定一系列路径的哈希值。源和目标中都类似。

- 我们判断一个文件是新的、重命名的还是多余的。

- 我们复制、移动或删除文件以匹配源。

请记住，我们希望为每个职责找到简化的抽象。这样我们就可以隐藏杂乱的细节，以便专注于有趣的逻辑。

!!! note
    在本章中，我们正在将一些复杂的代码重构为更易于测试的结构，通过识别需要完成的单独任务，并将每个任务分配给一个明确定义的参与者，类似于`duckduckgo`示例的方式。

对于步骤 1 和 2，我们已经直观地开始使用抽象，即哈希到路径的字典。您可能已经在想，“为什么不为目标文件夹和源文件夹建立一个字典，然后我们只需比较两个字典？”这似乎是抽象文件系统当前状态的好方法：

```
source_files = {'hash1': 'path1', 'hash2': 'path2'}
dest_files = {'hash1': 'path1', 'hash2': 'pathX'}
```

那么从步骤 2 转到步骤 3 怎么样？我们如何抽象出实际的移动/复制/删除文件系统交互？

我们将在这里使用一个技巧，本书后面会大规模使用这个技巧。我们将把我们想要做什么与如何去做分开。我们将让我们的程序输出一个如下所示的命令列表：

```
("COPY", "sourcepath", "destpath"),
("MOVE", "old", "new"),
```

现在我们可以编写仅使用两个文件系统字典作为输入的测试，并且我们期望表示操作的字符串元组列表作为输出。

我们不会说“给定这个实际的文件系统，当我运行我的函数时，检查发生了哪些操作”，而是说“给定这个文件系统的抽象，将会发生哪些文件系统操作的抽象？”

``` py title='简化测试中的输入和输出（test_sync.py）'
 def test_when_a_file_exists_in_the_source_but_not_the_destination():
        source_hashes = {'hash1': 'fn1'}
        dest_hashes = {}
        expected_actions = [('COPY', '/src/fn1', '/dst/fn1')]
        ...

    def test_when_a_file_has_been_renamed_in_the_source():
        source_hashes = {'hash1': 'fn1'}
        dest_hashes = {'hash1': 'fn2'}
        expected_actions == [('MOVE', '/dst/fn2', '/dst/fn1')]
        ...
```


## <font color='red'>实现我们选择的抽象</font>

这一切都很好，但是我们实际上如何编写这些新测试，以及如何改变我们的实现以使其正常工作？

我们的目标是隔离系统中的聪明部分，并能够在无需设置真实文件系统的情况下对其进行全面测试。我们将创建一个不依赖外部状态的代码“核心”，然后查看当我们从外部世界为其提供输入时它会如何响应（这种方法被Gary Bernhardt称为[函数核心，命令式外壳](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell)，或FCIS）。

让我们首先将代码拆分，将状态部分与逻辑分开。

我们的顶级函数几乎不包含任何逻辑；它只是一系列必要的步骤：收集输入，调用我们的逻辑，应用输出：

``` py title='将代码分成三部分（sync.py）'
def sync(source, dest):
    # imperative shell step 1, gather inputs
    source_hashes = read_paths_and_hashes(source)  #(1)
    dest_hashes = read_paths_and_hashes(dest)  #(1)

    # step 2: call functional core
    actions = determine_actions(source_hashes, dest_hashes, source, dest)  #(2)

    # imperative shell step 3, apply outputs
    for action, *paths in actions:
        if action == "COPY":
            shutil.copyfile(*paths)
        if action == "MOVE":
            shutil.move(*paths)
        if action == "DELETE":
            os.remove(paths[0])

```

1. 这是我们分离出来的第一个函数`read_paths_and_hashes()`，它隔离了我们应用程序的I/O部分。
2. 在这里我们划分出功能核心，即业务逻辑。

现在，构建路径和哈希字典的代码变得非常容易编写：

``` py title='仅执行 I/O 的函数（sync.py）'
def read_paths_and_hashes(root):
    hashes = {}
    for folder, _, files in os.walk(root):
        for fn in files:
            hashes[hash_file(Path(folder) / fn)] = fn
    return hashes
```

`determine_actions()`函数将是我们业务逻辑的核心，它表示：“给定这两组哈希和文件名，我们应该复制/移动/删除什么？”它接受简单的数据结构并返回简单的数据结构：

``` py title='仅执行业务逻辑的函数（sync.py）'
def determine_actions(source_hashes, dest_hashes, source_folder, dest_folder):
    for sha, filename in source_hashes.items():
        if sha not in dest_hashes:
            sourcepath = Path(source_folder) / filename
            destpath = Path(dest_folder) / filename
            yield "COPY", sourcepath, destpath

        elif dest_hashes[sha] != filename:
            olddestpath = Path(dest_folder) / dest_hashes[sha]
            newdestpath = Path(dest_folder) / filename
            yield "MOVE", olddestpath, newdestpath

    for sha, filename in dest_hashes.items():
        if sha not in source_hashes:
            yield "DELETE", dest_folder / filename
```

我们的测试现在直接作用于`determine_actions()`函数：

``` py title='更好看的测试（test_sync.py）'
def test_when_a_file_exists_in_the_source_but_not_the_destination():
    source_hashes = {"hash1": "fn1"}
    dest_hashes = {}
    actions = determine_actions(source_hashes, dest_hashes, Path("/src"), Path("/dst"))
    assert list(actions) == [("COPY", Path("/src/fn1"), Path("/dst/fn1"))]


def test_when_a_file_has_been_renamed_in_the_source():
    source_hashes = {"hash1": "fn1"}
    dest_hashes = {"hash1": "fn2"}
    actions = determine_actions(source_hashes, dest_hashes, Path("/src"), Path("/dst"))
    assert list(actions) == [("MOVE", Path("/dst/fn2"), Path("/dst/fn1"))]
```

因为我们已经将程序的逻辑（用于识别变化的代码）与 I/O 的低级细节分离开来，所以我们可以轻松地测试代码的核心。

通过这种方法，我们已经从测试我们的主入口函数`sync()`，转变为测试较低级的函数`determine_actions()`。您可能认为这样做很好，因为现在`sync()`函数变得非常简单。或者您可能决定保留一些集成/验收测试来测试`sync()`函数。但还有另一种选择，即修改`sync()`函数，以便对其进行单元测试和端到端测试；这是被Bob称之为端到端测试。


### <font color='red'>用Fakes和依赖注入进行端到端测试</font>

当我们开始编写新系统时，我们通常首先关注核心逻辑，并通过直接单元测试来驱动它。不过，在某些时候，我们希望一起测试系统的更大块

我们可以回到端到端测试，但这些测试仍然像以前一样难以编写和维护。相反，我们经常编写测试来调用整个系统，但模拟 I/O，有点像端到端：

``` py title='显式依赖项（sync.py）'
def sync(source, dest, filesystem=FileSystem()):  #(1)
    source_hashes = filesystem.read(source)  #(2)
    dest_hashes = filesystem.read(dest)  #(2)

    for sha, filename in source_hashes.items():
        if sha not in dest_hashes:
            sourcepath = Path(source) / filename
            destpath = Path(dest) / filename
            filesystem.copy(sourcepath, destpath)  #(3)

        elif dest_hashes[sha] != filename:
            olddestpath = Path(dest) / dest_hashes[sha]
            newdestpath = Path(dest) / filename
            filesystem.move(olddestpath, newdestpath)  #(3)

    for sha, filename in dest_hashes.items():
        if sha not in source_hashes:
            filesystem.delete(dest / filename)  #(3)
```

1. 我们的顶级函数现在公开了一个新的依赖项，即一个`FileSystem`。
2. 我们调用`filesystem.read()`来生成文件字典。
3. 我们调用FileSystem的`.copy()`、`.move()`和`.delete()`方法来应用我们检测到的变化。

!!! tip
    虽然我们使用了依赖注入，但没有必要定义抽象基类或任何形式的显式接口。在本书中，我们经常展示ABC，因为我们希望它们能帮助您理解抽象是什么，但它们并不是必需的。Python 的动态特性意味着我们始终可以依赖鸭子类型。

我们抽象的`FileSystem`实际上（默认）实现执行了真正的I/O：

``` py title='真正的依赖项（sync.py）'
class FileSystem:

    def read(self, path):
        return read_paths_and_hashes(path)

    def copy(self, source, dest):
        shutil.copyfile(source, dest)

    def move(self, source, dest):
        shutil.move(source, dest)

    def delete(self, dest):
        os.remove(dest)
```

但是假的依赖是围绕我们选择的抽象的包装器，而不是进行真正的I/O：

``` py title='使用 DI 进行测试'
class FakeFilesystem:
    def __init__(self, path_hashes):  #(1)
        self.path_hashes = path_hashes
        self.actions = []  #(2)

    def read(self, path):
        return self.path_hashes[path]  #(1)

    def copy(self, source, dest):
        self.actions.append(('COPY', source, dest))  #(2)

    def move(self, source, dest):
        self.actions.append(('MOVE', source, dest))  #(2)

    def delete(self, dest):
        self.actions.append(('DELETE', dest))  #(2)
```

1. 我们使用我们选择表示文件系统状态的抽象来初始化我们的伪文件系统：哈希到路径的字典。。
2. `.actions`只是将记录附加到列表中，以便我们以后可以检查它。这意味着我们的测试替身既是“假的”又是“间谍”。

因此，现在我们的测试可以作用于真实的顶层`sync()`入口点，但它们由`FakeFileSystem()`实现。就其设置和断言而言，它们最终看起来与我们直接针对功能核心`determine_actions()`函数进行测试时编写的代码非常相似：

``` py title='使用 DI 进行测试'
def test_when_a_file_exists_in_the_source_but_not_the_destination():
    fakefs = FakeFilesystem({
        '/src': {"hash1": "fn1"},
        '/dst': {},
    })
    sync('/src', '/dst', filesystem=fakefs)
    assert fakefs.actions == [("COPY", Path("/src/fn1"), Path("/dst/fn1"))]


def test_when_a_file_has_been_renamed_in_the_source():
    fakefs = FakeFilesystem({
        '/src': {"hash1": "fn1"},
        '/dst': {"hash1": "fn2"},
    })
    sync('/src', '/dst', filesystem=fakefs)
    assert fakefs.actions == [("MOVE", Path("/dst/fn2"), Path("/dst/fn1"))]
```

这种方法的优点是我们的测试作用于生产代码所使用的完全相同的函数。缺点是我们必须明确我们的状态组件并将它们传递出去。Ruby on Rails 的创建者 David Heinemeier Hansson 将此描述为“测试引起的设计损坏”。

无论哪种情况，我们现在都可以着手修复实现中的所有错误；枚举所有边缘情况的测试现在变得更加容易。


### <font color='red'>为什么不直接修复它呢？</font>

此时，您可能会抓耳挠腮，想着“为什么不直接使用`mock.patch`节省力气呢？”

在本书中，我们在本书和生产代码中都避免使用`mocks`。我们不会卷入一场战争，但我们的直觉是，`mocking`框架，特别是`monkeypatching`，是一种代码异味。

相反，我们希望明确识别代码库中的职责，并将这些职责分离为小的、集中的对象，以便于用测试替身进行替换

!!! note
    您可以在 [08.事件和消息总线](./l.Events%20and%20the%20Message%20Bus.md) 中看到一个例子，在这个例子中，我们使用了`mock.patch()`来替换一个发送电子邮件的模块，但最终我们在 [13.依赖注入](./q.Dependency%20Injection.md) 中用显示依赖替换了它。

我们的偏好有三个密切相关的原因：

- 修补您正在使用的依赖项可以对代码进行单元测试，但这对改进设计没有任何作用。使用`mock.patch()`不会让您的代码与`--dry-run`标志一起工作，也不会帮助您针对 FTP 服务器运行。为此，您需要引入抽象。
- 使用mocks的测试往往与代码库的实现细节更加耦合。这是因为mock验证了事物之间的交互：我们是否使用了正确的参数调用了`shutil.copy`？在我们的经验中，代码和测试之间的这种耦合往往会使测试变得更加脆弱。
- 过度使用mocks会导致复杂的测试套件，无法解释代码。

!!! note
    可测试性设计实际上意味着可扩展性设计。我们牺牲了一点复杂性，换来更简洁的设计，以适应新的用例。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>Mocks与Fakes；传统风格与伦敦学派TDD</strong></p>
<p>以下是关于mock和fake之间区别的简短且略微简单的定义：</p>

<ul>
    <li>mock用于验证某物如何使用；它们具有像<code>assert_called_once_with()</code>这样的方法。它们与伦敦学派 TDD 相关。</li>
    <li>fake是被替换对象的有效实现，但它们仅用于测试。它们不会在“现实生活中”发挥作用；我们的内存存储库就是一个很好的例子。但您可以使用它们对系统的最终状态而不是沿途的行为做出断言，因此它们与经典风格的 TDD 相关。</li>
</ul>
<p>在这里，我们稍微混淆了mock与spies，以及fake与stub，您可以在马丁·福勒（Martin Fowler）关于这个主题的经典文章中阅读到更长、更正确的答案，名为"<a src='https://martinfowler.com/articles/mocksArentStubs.html'>Mocks Aren’t Stubs</a>"。</p>
<p>可能也并不有助于理解的是，unittest.mock 提供的 MagicMock 对象严格来说并不是mock；如果说它们是什么，它们更像是spies。但它们也经常被用作stubs或dummies。在这里，我们保证我们现在已经结束了对测试替身术语的吹毛求疵。</p>
<p>那么伦敦学派和传统 TDD 有什么区别呢？您可以在我们刚刚引用的Martin Fowler文章中，以及<a src='https://softwareengineering.stackexchange.com/questions/123627/what-are-the-london-and-chicago-schools-of-tdd'>softwareengineering网站</a>上，了解更多关于这两种学派的内容，但在本书中，我们更倾向于经典风格。我们喜欢在设置和断言中围绕状态来构建测试，并且我们喜欢在尽可能高的抽象级别上工作，而不是检查中间协作者的行为。</p>
<p>在[ki]</p>
</div>

我们认为 TDD 首先是一种设计实践，其次才是测试实践。测试记录了我们的设计选择，并在我们长时间离开后重新查看代码时向我们解释系统。

使用过多模拟的测试会因设置代码而变得难以承受，而这些设置代码会隐藏我们关心的故事。

Steve Freeman在他的演讲"Test-Driven Development"中提供了一个很好的过度mock的例子。您也应该看一下我们尊敬的技术审阅员Ed Jung的PyCon演讲"Mocking and Patching Pitfalls"，该演讲也涉及mocks及其替代方法。

我们在推荐演讲的同时，请看一下Brandon Rhodes的精彩演讲"Hoisting Your I/O"。它实际上不是关于mock的，而是关于将业务逻辑与I/O解耦的一般问题，其中他使用了一个非常简单的说明性示例。

!!! tip
    在本章中，我们花了很多时间用单元测试取代端到端测试。但这并不意味着我们认为您永远不应该使用 E2E 测试！在本书中，我们将向您展示一些技巧，帮助您构建一个体面的测试金字塔，其中包含尽可能多的单元测试，以及让您充满信心所需的最少数量的 E2E 测试。

<div style="border:1px solid #e0e0dc;background:#f8f8f7;padding:15px;">
<p align='center'><strong style='font-size:1.6em;color:#186d7a'>那么我们在本书中使用哪一个呢？函数式还是面向对象的组合？</strong></p>
<p>两者都有。我们的领域模型完全没有依赖关系和副作用，所以这就是我们的功能核心。我们围绕它构建的服务层（在[4.服务层](./g.Flask%20API%20and%20Service%20Layer.md)中）使我们能够从边缘到边缘驱动系统，并且我们使用依赖注入为这些服务提供有状态组件，因此我们仍然可以对它们进行单元测试。</p>

<p>请查看[13.依赖注入](./q.Dependency%20Injection.md)以了解更多关于如何使我们的依赖注入更加明确和集中的探索。</p>
</div>


## <font color='red'>总结</font>

我们会在本书中一次又一次地看到这个想法：通过简化业务逻辑和混乱的 I/O 之间的接口，我们可以使我们的系统更易于测试和维护。找到正确的抽象很棘手，但这里有一些启发式方法和问题要问自己：

- 我是否可以选择一个熟悉的 Python 数据结构来表示混乱系统的状态，然后尝试想象一个可以返回该状态的单个函数？

- 将“什么”与“如何”分开：我是否可以使用数据结构或 DSL 来表示我想要发生的外部效果，而与我计划如何实现它们无关？

- 我可以在哪里在我的系统之间划一条线，我可以在哪里挖出一条接缝 来粘贴那个抽象？

- 将事物划分为具有不同职责的组件的合理方法是什么？我可以明确哪些隐含的概念？

- 依赖关系是什么，核心业务逻辑是什么？

实践可以减少不完美！现在回到我们的常规编程中...