### 定义

> 将一个复杂对象的构建与它的表示分离，使用同样的构建过程可以创建不同的表示。

### 角色

Product、Builder

### 为什么要使用Builder模式？

直接使用构造函数或者配合 set 方法就能创建对象，为什么还需要建造者模式来创建呢？

假设我们需要定义一个资源配置类``ResourcePoolConfig``，它有一个**必填**的`name`参数和**可选**的`maxTotal`、`maxIdle`、`minIdle`，编码如下:

```java
public class ResourcePoolConfig {
  private static final int DEFAULT_MAX_TOTAL = 8;
  private static final int DEFAULT_MAX_IDLE = 8;
  private static final int DEFAULT_MIN_IDLE = 0;

  private String name;
  private int maxTotal = DEFAULT_MAX_TOTAL;
  private int maxIdle = DEFAULT_MAX_IDLE;
  private int minIdle = DEFAULT_MIN_IDLE;

  public ResourcePoolConfig(String name, Integer maxTotal, Integer maxIdle, Integer minIdle) {
    if (StringUtils.isBlank(name)) {
      throw new IllegalArgumentException("name should not be empty.");
    }
    this.name = name;

    if (maxTotal != null) {
      if (maxTotal <= 0) {
        throw new IllegalArgumentException("maxTotal should be positive.");
      }
      this.maxTotal = maxTotal;
    }

    if (maxIdle != null) {
      if (maxIdle < 0) {
        throw new IllegalArgumentException("maxIdle should not be negative.");
      }
      this.maxIdle = maxIdle;
    }

    if (minIdle != null) {
      if (minIdle < 0) {
        throw new IllegalArgumentException("minIdle should not be negative.");
      }
      this.minIdle = minIdle;
    }
  }
  //...省略getter方法...
}
```

现在，ResourcePoolConfig 只有 4 个可配置项，对应到构造函数中，也只有 4 个参数。但是，如果可配置项逐渐增多，变成了 8 个、10 个，甚至更多，那继续沿用现在的设计思路，构造函数的参数列表会变得很长，代码在可读性和易用性上都会变差：

```java
// 参数太多，导致可读性差、参数可能传递错误
ResourcePoolConfig config = new ResourcePoolConfig("dbconnectionpool", 16, null, 8, null, false , true, 10, 20，false， true);
```

我们可以使用`setter`解决这个问题，构造函数中只需要传递必填的参数，其他可选的参数使用`setter`方法设置：

```java
public class ResourcePoolConfig {
  private static final int DEFAULT_MAX_TOTAL = 8;
  private static final int DEFAULT_MAX_IDLE = 8;
  private static final int DEFAULT_MIN_IDLE = 0;

  private String name;
  private int maxTotal = DEFAULT_MAX_TOTAL;
  private int maxIdle = DEFAULT_MAX_IDLE;
  private int minIdle = DEFAULT_MIN_IDLE;
  
  public ResourcePoolConfig(String name) {
    if (StringUtils.isBlank(name)) {
      throw new IllegalArgumentException("name should not be empty.");
    }
    this.name = name;
  }

  public void setMaxTotal(int maxTotal) {
    if (maxTotal <= 0) {
      throw new IllegalArgumentException("maxTotal should be positive.");
    }
    this.maxTotal = maxTotal;
  }

  public void setMaxIdle(int maxIdle) {
    if (maxIdle < 0) {
      throw new IllegalArgumentException("maxIdle should not be negative.");
    }
    this.maxIdle = maxIdle;
  }

  public void setMinIdle(int minIdle) {
    if (minIdle < 0) {
      throw new IllegalArgumentException("minIdle should not be negative.");
    }
    this.minIdle = minIdle;
  }
  //...省略getter方法...
}

//使用
ResourcePoolConfig config = new ResourcePoolConfig("dbconnectionpool");
config.setMaxTotal(16);
config.setMaxIdle(8);
```

到这里我们仍然没有用到Builder模式。那是因为我们没有必要用它。现在试考虑以下情景：

- 必填参数有很多。如果我们把必填参数放到构造函数中，参数列表会冗长；如果放到setter函数中，必填项校验的逻辑要放在哪里呢？
- 我们希望对象是不可变的。对象创建完成后不能对其属性做修改，这时候我们就不能暴露setter方法。
- 配置项之间有依赖关系。比方说设置了一个必须同时设置其他几个；再比如说几个配置项之间有逻辑关系。
- 类构建出来就是有效的。不能因为没有设置某些属性导致类表现为不正确或不合逻辑。

这个时候，构造函数和setter就不中用了。我们把构造函数置为**私有**的，参数校验集中放在`Builder`类的`build`方法中：

```java
public class ResourcePoolConfig {
  private String name;
  private int maxTotal;
  private int maxIdle;
  private int minIdle;

  private ResourcePoolConfig(Builder builder) {
    this.name = builder.name;
    this.maxTotal = builder.maxTotal;
    this.maxIdle = builder.maxIdle;
    this.minIdle = builder.minIdle;
  }
  //...省略getter方法...

  //我们将Builder类设计成了ResourcePoolConfig的内部类。
  //我们也可以将Builder类设计成独立的非内部类ResourcePoolConfigBuilder。
  public static class Builder {
    private static final int DEFAULT_MAX_TOTAL = 8;
    private static final int DEFAULT_MAX_IDLE = 8;
    private static final int DEFAULT_MIN_IDLE = 0;

    //这也算是一个缺点，属性要在原始类和Builder类中都写一遍
    private String name;
    private int maxTotal = DEFAULT_MAX_TOTAL;
    private int maxIdle = DEFAULT_MAX_IDLE;
    private int minIdle = DEFAULT_MIN_IDLE;

    public ResourcePoolConfig build() {
      // 校验逻辑放到这里来做，包括必填项校验、依赖关系校验、约束条件校验等
      if (StringUtils.isBlank(name)) {
        throw new IllegalArgumentException("...");
      }
      if (maxIdle > maxTotal) {
        throw new IllegalArgumentException("...");
      }
      if (minIdle > maxTotal || minIdle > maxIdle) {
        throw new IllegalArgumentException("...");
      }

      //返回原始类对象
      return new ResourcePoolConfig(this);
    }

    public Builder setName(String name) {
      if (StringUtils.isBlank(name)) {
        throw new IllegalArgumentException("...");
      }
      this.name = name;
      //setter方法中返回this 以便链式调用
      return this;
    }

    public Builder setMaxTotal(int maxTotal) {
      if (maxTotal <= 0) {
        throw new IllegalArgumentException("...");
      }
      this.maxTotal = maxTotal;
      return this;
    }

    public Builder setMaxIdle(int maxIdle) {
      if (maxIdle < 0) {
        throw new IllegalArgumentException("...");
      }
      this.maxIdle = maxIdle;
      return this;
    }

    public Builder setMinIdle(int minIdle) {
      if (minIdle < 0) {
        throw new IllegalArgumentException("...");
      }
      this.minIdle = minIdle;
      return this;
    }
  }
}

// 这段代码会抛出IllegalArgumentException，因为minIdle>maxIdle
ResourcePoolConfig config = new ResourcePoolConfig.Builder()
        .setName("dbconnectionpool")
        .setMaxTotal(16)
        .setMaxIdle(10)
        .setMinIdle(12)
        .build();
```

### Kotlin中的Builder

Kotin、Dart的函数都支持**默认参数和命名参数**，这使得对象创建比Java简洁的多：

```kotlin
class Computer(
    val cpu: String = "英特尔",
    val ram: String = "金士顿",
    val usbCount: Int = 2,
    val keyboard: String = "罗技",
    val display: String = "京东方"
) {
    override fun toString(): String {
        return "Computer(cpu='$cpu', ram='$ram', usbCount=$usbCount, keyboard='$keyboard', display='$display')"
    }
}

//调用
val computer= Computer(ram="海力士",keyboard = "双飞燕")
```

即便如此，遇到上边提到的几种情景，它们也是干瞪眼。

因为Kotlin支持**接收器函数**(相当于函数中能获得当前接收器的引用)，Builder模式实现起来可以比Java优雅的多：

```kotlin
//构造函数中接收Builder 做赋值操作
class Computer private constructor(...){

    companion object {
      	//这么理解：build方法接收一个无参、无返回的lambda，并将自身对象传入到这个lambda中
      	//这意味着我可以在这个lambda中调用Builder的setter
        inline fun build(block: Builder.() -> Unit) = Builder().apply(block).build()
    }

    class Builder {
        //... setter
        fun build() = Computer(this)
    }
}

//使用
val computer = Computer.build {
         setCup("AMD")
         setRam("海力士")
         setDisplay("三星")
         setUsb(3)
         setKeyboard("双飞燕")
 }
```

当然，你也可以像编写Java那样写Kotlin，清晰明了是最重要的。只不过，Kotlin中有一些好玩的语法糖很有意思，有意思本来就是一件有意思的事情。

### 与工厂模式的区别?

> 顾客走进一家餐馆点餐，我们利用工厂模式，根据用户不同的选择，来制作不同的食物，比如披萨、汉堡、沙拉。对于披萨来说，用户又有各种配料可以定制，比如奶酪、西红柿、起司，我们通过建造者模式根据用户选择的不同配料来制作披萨。

