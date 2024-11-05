* Refactored payment feature with Strategy Design Pattern, enabling province-specific exchange rates and seamless policy updates. Improved system extensibility for multi-provincial payment handling, cutting new policy implementation time by 50%.
# 情景
* 我在一个保险公司担任软件开发工程师。最初系统只支持安大略省和卑诗省的业务，随着公司扩展到曼尼托巴等其他省份，我发现原有的if-else结构使得每增加一个省份都需要修改核心代码。因此，我使用策略模式重构了税率计算模块，不仅提高了代码的可维护性，也为后续业务扩展提供了更好的支持。
## 解决了原有问题:
* 原来的if-else结构使得每添加一个新省份都需要修改核心代码
* 代码维护困难,容易出错
* 违反了开闭原则

## 策略模式的优势:
* 遵循开闭原则：添加新省份只需创建新的策略类,无需修改现有代码
* 每个省份的税率计算逻辑被封装在独立的类中,便于维护
* 通过策略工厂统一管理策略的创建
* 提高了代码的可测试性

## 具体改进:
* 将税率计算逻辑从原来的if-else抽象为策略接口
* 每个省份实现自己的计算策略
* 添加策略工厂简化客户端使用
* 提供了清晰的错误处理机制

## 业务价值:
* 支持业务快速扩展到新省份
* 降低了维护成本
* 减少了代码错误风险
* 提高了代码重用性

```java
interface BasePayment {
  double pay(id, amount);
}
//close modification
class PaymentDepartment {
  //容器承载
  BasePayment payment;
  public paymentDepartment (BasePayment payment) {
    this.paymnet = payment;
  }
 	public double processPayment (id, amount) {
    payment.pay(id,amount);
  }
}
//open exension
class ONPayment implemnets BasePayment {
  @Override
  double pay(id, amount) {
    return 1.3 * amount;
  }
}
```