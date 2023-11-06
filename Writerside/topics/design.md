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

## 代理模式
> 为了方便访问某些资源，使对象类更加易用，从而在操作上使用的代理服务

- 主体(Subject)：Subject角色定义了使Proxy角色和RealSubject角色之间具有一致性的接口
- 代理人(Proxy)：Proxy角色会尽量处理来自Client角色的请求。只有当自己不能处理的时候，它才会将工作交给RealSubject
  角色。Proxy角色只有在必要的时候才会生成RealSubject。Proxy角色实现了在Subject角色中定义的接口
- 实际的主体(RealSubject)：RealSubject角色会在Proxy角色无法胜任工作时出场，同样也会实现Subject角色中的接口
- 请求者(Client)：使用Proxy模式的角色，不包含在代理模式中

> 可以在代理类中实现一些轻量级的操作，等到真正要实现被代理类中的重量级操作的时候再去初始化被代理类

示例代码实现一个将文字打印出来的打印机功能
1. 定义主体接口
```Java
/**
 * 主体接口，保证代理类和实际主体的实现方法一致
 * @author yinrongjie
 * @date 2023/11/6
 * @name Printable
 */
public interface Printable {
    /**
     * 设置打印机名称
     * @param name
     */
    void setPrinterName(String name);

    /**
     * 获取打印机名称
     * @return
     */
    String getPrinterName();

    /**
     * 打印一个字符串
     * @param content
     */
    void print(String content);
}
```
2. 定义一个RealSubject角色
```Java
/**
 * RealSubject
 * @author yinrongjie
 * @date 2023/11/6
 * @name Printer
 */
@Slf4j
public class Printer implements Printable {
    private String name;

    public Printer() {
        // 表示生成这个Printer实例是一个非常耗时或者消耗资源的操作
        this.heavyJob("Printer实例生成中。。。");
    }

    public Printer(String name) {
        this.name = name;
        // 表示生成这个Printer实例是一个非常耗时或者消耗资源的操作
        this.heavyJob("Printer实例生成中。。。");
    }

    /**
     * 设置打印机名称
     *
     * @param name
     */
    @Override
    public void setPrinterName(String name) {
        this.name = name;
    }

    /**
     * 获取打印机名称
     *
     * @return
     */
    @Override
    public String getPrinterName() {
        return this.name;
    }

    /**
     * 打印一个字符串
     *
     * @param content
     */
    @Override
    public void print(String content) {
        log.info("This is print [{}]", this.name);
        log.info("Print content is [{}]", content);
    }

    /**
     * 表示要做一个非常重的操作
     * @param str
     */
    @SneakyThrows
    private void heavyJob(String str) {
        log.info(str);
        for (int i = 0; i < 10; ++i) {
            Thread.sleep(1000);
            log.info(".");
        }
        log.info("Heavy job finished.");
    }
}
```
3. 定义代理对象
```Java
/**
 * 代理对象
 * @author yinrongjie
 * @date 2023/11/6
 * @name PrintProxy
 */
public class PrintProxy implements Printable {
    /**
     * 代理对象本身的名称
     */
    private String name;

    /**
     * 被代理对象
     */
    private Printer real;

    public PrintProxy() {
    }

    public PrintProxy(String name) {
        this.name = name;
    }

    /**
     * 设置打印机名称
     *
     * @param name
     */
    @Override
    public synchronized void setPrinterName(String name) {
        if (real != null) {
            // 如果已经生成了被代理对象，那么给被代理对象赋值
            real.setPrinterName(name);
        }
        this.name = name;
    }

    /**
     * 获取打印机名称
     *
     * @return
     */
    @Override
    public String getPrinterName() {
        return this.name;
    }

    /**
     * 打印一个字符串
     *
     * @param content
     */
    @Override
    public void print(String content) {
        this.realize();
        this.real.print(content);
    }

    /**
     * 构造被代理对象
     */
    private synchronized void realize() {
        if (this.real == null) {
            this.real = new Printer(this.name);
        }
    }
}
```
4. 调用代理类的主方法
```Java
@Slf4j
public class ProxyMain {
    public static void main(String[] args) {
        Printable printer = new PrintProxy("Proxy printer");
        log.info("This is [{}] printer.", printer.getPrinterName());
        printer.setPrinterName("Proxy printer1");
        log.info("This is [{}] printer.", printer.getPrinterName());
        printer.print("This is print content.");
    }
}
```

### 静态代理
> 静态代理是指预先确定好了代理者和被代理者的关系，即代理类和被代理类的关系在编译期间就已经确定