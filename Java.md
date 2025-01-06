继承的思想：父类构造器完成父类属性的初始化,，子类的构造器完成子类属性的初始化。

```
public Computer(String cpu, int mem, int disk) | PC extends Computer | public PC(){} Computer缺少默认构造
```

super访问父类的属性/方法，不能访问private属性/方法。

访问父类构造器，super(参数列表) ，只能放在构造器的第一句。