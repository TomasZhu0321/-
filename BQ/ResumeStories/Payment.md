# Payment API
* Refactored payment feature with Strategy Design Pattern, enabling province-specific exchange rates and seamless policy updates. Improved system extensibility for multi-provincial payment handling, cutting new policy implementation time by 50%.

## 故事
* **Situation**: 公司的Payment feature因业务扩展需要根据不同省份进行不同的汇率计算和价格生成。
* **Task**: 实现新的省份价格计算模块，以适应业务扩展需求。
* **Action**: 
  * 发现之前的实现方式采用if-else结构，随着业务扩展出现了代码复杂度高、可维护性差的问题，对未来扩展也存在局限性。
  * 进行了广泛的技术调研，发现采用Strategy Design Pattern的方式能够有效解决现有问题并支持未来扩展。
  * 与Manager进行了开会讨论，present了新的设计方案，并且提出重构(Refactor)之前payment功能的必要性和潜在收益。
  * 对功能进行了重构，实现了新的计算模块，并通过全面的单元测试进行验证。
  * 使用Confluence编写了详细的文档，包括设计理念、使用方法和未来扩展指南。
* **Result**:
  * 新的模块设计使得添加新省份的价格计算逻辑变得简单快速，在接下来的3个月内，团队采用类似模式重构了2个其他核心模块。减少了implementation time by 50%.
  * 提高了整个系统的可扩展性和可维护性。
  * 因为我的ownership和主动性，获得了同事和领导的认可和信任。

## 技术知识点
* 策略模式，行为型设计模式
* 核心思想：定义一系列算法（支付策略），将每个算法封装起来，并使它们可以互相替换. 
* 实现：
  * 首先通过BasePayment接口定义了统一的支付行为，实现了代码的解耦，提高了系统的可扩展性和灵活性。
  * PaymentDepartment类作为上下文，持有策略对象，并提供统一的接口来处理来自controller的支付请求。
  * 具体的支付策略类（如ONPayment, BCPayment）实现BasePayment接口，各自处理特定的支付逻辑和汇率计算。
* 好处：
  * PaymentDepartment满足了Close to modification原则，不需要因新支付方式而修改。
  * 具体的支付类（如ONPayment, BCPayment）满足了Open to Extension原则，可以轻松添加新的支付方式而无需修改现有代码。
  * 支持在运行时动态切换支付策略，增加了系统的灵活性。
***
```java
interface BasePayment {
  pay (id, amount);
}
//Close to modification
// 定义client关注的interface
class PaymentDepartment {
  //持有策略对象
  BasePayment payment;
  public PaymentDepartment (BasePayment payment) {
    this.payment = payment;
  }
  //统一的接口，处理支付请求
  public double processPayment (id, amount) {
    return payment.pay(id, amount);
  }
}
//Open to Extension
class ONPayment implements BasePayment {
  @Override
  public double pay(id, amount);
}
class BCPayment implements BasePayment {
  @Override
  public double pay(id, amount);
}
```

