# 前提

本文的标题会被老一派的DDD（领域驱动设计）实践者们所诟病，但本文站在一个如下设计模式已经被广泛接受的前提下：

* 命令模式
* 事件驱动
* CQRS
* Saga

今天的DDD，早已不是2004年Eric Evans所描述的DDD了！2019年，Eric Evans在Explore DDD呼吁与会者们“改进”DDD的“语言描述”，以解决各种术语被错误理解，导致背后的思想与理念无法被正确传达的问题。前些日子听大家聊”领域模型（Domain Model）“时，才大家发现在敏捷开发、RUP、json框架，需求分析等不同领域，大家对“领域模型”这个词的理解竟然天差地别。“domain”和“model”这两个词在英文语境下的滥用导致了极其严重的误解。有人甚至提出DDD在国内被深刻误解的一大原因就是DDD这个名字，“领域驱动设计”这个翻译太广泛、太容易望文生义了，如果让他来编写领域驱动设计那本书，他一定会用一个谁都看不懂的英文单词来命名，逼大家去理解背后的思想，而不是停留在字面意思上。本文极为赞赏这种行为，并身体力行的进行了实践：本文将详细说明“领域服务（Domain Service）”这一概念为什么需要被清除出DDD的设计体系，并说明其背后的思想与理念应当被落地到何处。

# 案例一：银行转账

当用户故事是：A 向 B 转账 100 元时，代码通常会这样写：

```csharp
fromAccount.Debit(100);
toAccount.Credit(100);
```

这种写法的问题在于：

* Debit 是 A 的职责
* Credit 是 B 的职责
* 转账行为本身并不属于 A 或 B 的职责范围

为了处理这个问题，通常会引入一个领域服务（Domain Service）：

```csharp
class TransferService
{
    void Transfer(Account from, Account to, Money amount)
    {
        from.Debit(amount);
        to.Credit(amount);
    }
}
```

该处理方法，认为转账不是“账户的行为”，而是“账户之间的业务规则”。

# 案例二：电商下单

当用户故事是：用户下单购买商品时，需要确保用户没有被封禁，且商品有库存时，我们会发现：

* User 是一个聚合
* Inventory 是另一个聚合

两者互相独立，无法通过一个聚合的方法来完成下单行为。于是会引入一个领域服务：

```csharp
class OrderService
{
    void PlaceOrder(User user, Inventory inventory, OrderDetails details)
    {
        if (user.IsBanned()) throw new Exception("User is banned");
        if (!inventory.HasStock(details.ProductId)) throw new Exception("Out of stock");
        // 继续处理下单逻辑
    }
}
```

该处理方法，认为横跨多个聚合的业务规则应当被放在领域服务中。

# 案例三：复杂策略 / 算法

当用户故事是：根据不同的折扣策略计算商品价格时，我们会发现：

* 折扣策略可能会频繁变化
* 折扣策略本身没有状态，只是一种行为
* 折扣策略可能需要领域内部的多个聚合参与计算
* 折扣策略的确归属于领域层，应用层不应知晓其细节

于是会引入一个领域服务：

```csharp
class PricingService
{
    Money CalculatePrice(Product product, User user, Time time);
}
```

# 案例一的问题

Evans 写书时，并没有把 CQRS / Saga 作为默认前提。在他的语境下，一个用例 = 一个事务。因此，“转账”是一个不可分割的业务概念，它必须被放在一个地方处理。于是他引入了“领域服务”这个概念。

但在命令模式+事件驱动+Saga 的语境下，转账行为已经被拆分成了如下业务流程：

```text
CommandHandler:
  Load A
  A.Debit()
  Save A
  Publish Event

Saga:
  On A_Debited
  Load B
  B.Credit()
```

无论是命令处理器（Command Handler）还是流程编排器（Saga），都属于应用层的职责范畴。因此，“转账”毫无疑问是应用层的代码进行处理。

# 案例二的问题

这个所谓的 `OrderService` 实际上做的是应用层（Application Layer）该做的事情：编排（Orchestration）。

在命令模式下，`PlaceOrder` 是一个命令（Command）。处理这个命令的 Command Handler 负责加载相关聚合根，执行检查，并持久化结果。

```csharp
class PlaceOrderCommandHandler
{
    public void Handle(PlaceOrderCommand cmd)
    {
        var user = _userRepo.Load(cmd.UserId);
        if (user.IsBanned()) throw new Exception("...");
        
        var inventory = _inventoryRepo.Load(cmd.ProductId);
        if (!inventory.HasStock(cmd.Count)) throw new Exception("...");
        
        var order = Order.Create(user, inventory, cmd.Count);
        _orderRepo.Save(order);
    }
}
```

请注意，`user.IsBanned()` 和 `inventory.HasStock()` 依然是领域逻辑，它们留在各自的聚合中。`Order.Create` 也是领域逻辑。但是，**“先检查用户，再检查库存，最后创建订单”** 这个流程，是业务用例（Use Case）的逻辑，属于应用层。

强行引入一个领域服务来容纳这段逻辑，导致领域层变得臃肿且包含了不该有的流程控制。

# 案例三的问题

这个案例触及了面向对象编程（OOP）中最容易被误解的一点：将“行为”强行封装成“服务”。

如果折扣策略频繁变化，且作为领域的核心规则，那么它本身就应该被建模为一个对象，而不是一个服务。在这个场景下，我们应该使用策略模式（Strategy Pattern）或者将其建模为值对象。

```csharp
interface IPricingPolicy
{
    Money Calculate(Product product, User user);
}

class SeasonalPricingPolicy : IPricingPolicy { ... }
```

然后，通过双分派（Double Dispatch）或者纯参数传递的方式交给聚合使用：

```csharp
// 方式 A：策略作为参数
order.CalculateFinalPrice(new SeasonalPricingPolicy());

// 方式 B：工厂/构建器中使用策略
var order = OrderFactory.Create(product, user, new SeasonalPricingPolicy());
```

将它命名为 `PricingService` 是一种偷懒的行为。Service 这个后缀通常暗示着它可能包含副作用（如数据库查询、API调用），或者它是单例的、无状态的。但在这里，我们实际上想要表达的是一个**核心领域概念**——“定价规则”。

把它显式地建模为 `Policy`（策略）、`Rule`（规则）或者 `Calculator`（计算器，作为纯函数），远比叫它 `Service` 要精准得多。在领域模型中，我们应当极力避免使用泛泛的 `Service`、`Manager`、`Helper` 这种毫无业务含义的后缀。

# 总结

通过上述分析，我们可以得出结论：

领域服务（Domain Service）这一概念在以命令模式、事件驱动、CQRS、Saga 为基础的现代 DDD 实践中已经变得多余，甚至有害。它掩盖了应用层和领域层之间的职责划分，将过多的业务逻辑挤压进了领域层。

请以极其严肃的态度对待任何还在大肆教学如何使用“领域服务”的DDD资料——它落伍了！