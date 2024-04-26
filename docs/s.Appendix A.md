# 附录A：摘要图表


以下是我们在本书结尾处的架构：

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_aa01.png){ align=center }
    <figcaption><font size=2></font></figcaption>
</figure>

表1.我们的架构的组件及其功能 概括了每个模式及其功能。

<font color='#186d7a'>表1.我们的架构的组件及其功能</font>

<table>
    <tr>
        <th>层</th>
        <th>组件</th>
        <th>描述</th>
    </tr>
    <tr>
        <td rowspan="5"><strong>领域</strong><br>定义业务逻辑</td>
        <td>实体</td>
        <td>域对象，其属性可能会发生变化，但随着时间的推移具有可识别的身份</td>
    </tr>
    <tr>
        <td>值对象</td>
        <td>一个不可变的域对象，其属性完全定义它。它可以与其他相同的对象互换</td>
    </tr>
    <tr>
        <td>聚合</td>
        <td>关联对象的集群，我们将其视为数据更改的一个单元。定义并强制执行一致性边界</td>
    </tr>
    <tr>
        <td>事件</td>
        <td>代表发生的事情</td>
    </tr>
    <tr>
        <td>命令</td>
        <td>表示系统应该执行的工作</td>
    </tr>
    <tr>
        <td rowspan="3">服务层 <br>定义系统应执行的工作并协调不同的组件</td>
        <td>处理程序</td>
        <td>接收命令或事件并执行需要发生的操作。</td>
    </tr>
    <tr>
        <td>工作单元</td>
        <td>围绕数据完整性的抽象。每个工作单元代表一个原子更新。使存储库可用。跟踪检索到的聚合上的新事件</td>
    </tr>
    <tr>
        <td>消息总线（内部）</td>
        <td>通过将命令和事件路由到适当的处理程序来处理它们</td>
    </tr>
    <tr>
        <td rowspan="2">适配器（次级）<br>从我们的系统到外部世界（I/O）的接口的具体实现。</td>
        <td>存储库</td>
        <td>围绕持久存储的抽象。每个聚合都有自己的存储库</td>
    </tr>
    <tr>
        <td>事件发布者</td>
        <td>将事件推送到外部消息总线上</td>
    </tr>
    <tr>
        <td rowspan='2'><strong>入口点（主适配器）</strong><br>将外部输入转换为对服务层的调用。|网页|接收网络请求并将其转换为命令，传递给内部消息总线</td>
        <td>网页</td>
        <td>接收网络请求并将其转换为命令，传递给内部消息总线</td>
    </tr>
    <tr>
        <td>事件消费者</td>
        <td>从外部消息总线读取事件并将其转换为命令，传递给内部消息总线</td>
    </tr>
    <tr>
        <td>不适用</td>
        <td>外部消息总线（消息代理）</td>
        <td>不同服务通过事件进行相互通信所使用的基础设施</td>
    </tr>
</table>