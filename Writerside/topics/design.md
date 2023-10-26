# 设计模式

示例代码github地址：https://github.com/Yin-RJ/design-pattern.git

## 装饰器模式
> 装饰器模式的核心是在不改变原有类的基础上给类新增功能，解决直接继承时因功能的不断横向扩展导致子类膨胀的问题，
> 而使用装饰器模式比直接继承更加灵活，同时也不再需要维护子类

- 抽象构件角色(Component)：定义抽象接口
- 具体构件角色(ConcreteComponent)：实现抽象接口，可以是一组
- 装饰角色(Decorator)：定义抽象类并继承接口中的方法，保证一致性
  - 继承了处理接口
  - 提供了构造函数，构造函数的入参是继承的接口类的实现
  - 覆盖了方法
- 具体装饰角色(ConcreteDecorator)：扩展装饰具体的实现逻辑

考虑现在有一个做咖啡的场景，我可以做原味的咖啡，同样我可以在咖啡里加奶或者加糖，也可以都加
1. 定义抽象构件角色，提供一个做咖啡的接口
```Java
public interface ICoffee {
    void makeCoffee();
}
```
2. 定义一个具体构件角色，也就是要被装饰的原味咖啡
```Java
@Slf4j
public class OriginalCoffee implements ICoffee {
    @Override
    public void makeCoffee() {
        log.info("This is an original coffee.");
    }
}
```
3. 定义装饰角色，装饰角色需要实现抽象构件角色的方法，同时需要持有抽象构件角色的对象
```Java
public abstract class CoffeeDecorator implements ICoffee {
    private final ICoffee coffee;

    public CoffeeDecorator(ICoffee coffee) {
        this.coffee = coffee;
    }

    @Override
    public void makeCoffee() {
        this.coffee.makeCoffee();
    }
}
```
4. 定义具体装饰角色，这里我们定义一个加奶的装饰角色和一个加糖的装饰角色
```Java
@Slf4j
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(ICoffee coffee) {
        super(coffee);
    }

    @Override
    public void makeCoffee() {
        super.makeCoffee();
        addMilk();
    }

    private void addMilk() {
        log.info("Add milk.");
    }
}

@Slf4j
public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(ICoffee coffee) {
        super(coffee);
    }

    @Override
    public void makeCoffee() {
        super.makeCoffee();
        addSugar();
    }

    private void addSugar() {
        log.info("Add sugar.");
    }
}
```