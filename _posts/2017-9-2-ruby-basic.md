---
layout: post
title: Ruby入门
category: notes
tags: Ruby 元编程
published: true
---

* 文章目录
{:toc}

工作进入了Ruby项目组，入坑Ruby。都说Ruby简单，但是与其他静态语言差异很大，初次接触会有很多比较困惑的地方。写下这篇笔记帮助我梳理一些我认为比较新鲜的地方。

## 作用域

### 作用域门
Ruby中没有嵌套的作用域，进入一个新的作用域时，前一个作用域会关闭。有三个关键字来充当作用域门(Scope Gate):class、module和def。其中def定义的代码不会立即执行，而前两者中的代码会立即执行。

{% highlight ruby %}
v1 = 1
class MyClass
  v2 = 2
  p local_variables
  def my_method
    v3 = 3
    p local_variables
  end
  p local_variables
end

obj = MyClass.new
obj.my_method
p local_variables
{% endhighlight %}

上面的代码中存在顶层、MyClass和my_method三个作用域，互不可见。

### 作用域穿越

如果想让一个变量穿越作用域，需要用到代码块，代码块可以理解为匿名函数。把一个代码块传给Class.new，就可以穿越作用域。
{% highlight ruby %}
v1 = 2
MyClass = Class.new do
  p v1.to_s + "in MyClass"
  define_method :my_method do
    p v1.to_s + "in my_method"
  end
end
MyClass.new.my_method
{% endhighlight %}
上面的代码中，三个作用域好像被“挤压”在了一起，称为扁平作用域。有了这样的特性，我们可以进一步控制作用域。

### 闭包
{% highlight ruby %}
def make_stack
  index = -1
  data = {}
  define_method :push  do |ele|
    index = index + 1
    data[index] = ele
  end
  define_method :pop  do
    return nil if index == -1
    index = index - 1
    return data.delete(index+1)   
  end
  define_method :show do
    data.each {|e| p e}
  end
end
s = make_stack
s.push("a")
s.push("b")
s.pop()
s.show
{% endhighlight %}
上面的代码中，我们将变量data封装在了make_stack这个作用域中，并在push、pop和show这三个方法中共享，这种技巧叫做共享作用域。这段代码看起来很像一个类，其实这是利用闭包(Ruby中的代码块)来实现面向对象抽象的一种方式，闭包是函数式编程语言的一个重要特性。

函数式是不同于面向对象的另外一种编程思想，有一句话可以总结这两者的特点：**对象是附有行为的数据，而闭包是附有数据的行为**。关于函数式和闭包，又是另外一个很大的话题，就不展开了。

## 对象模型
Ruby的对象模型和其他静态语言不一样，有一些新的概念。**Ruby是纯面向对象，类也是对象。**
### 单件方法和单件类
单件方法是Ruby很有意思的一个特性，它可以让对象拥有自己的方法，先上一段代码说明一下：
{% highlight ruby %}
class MyClass; end
v1 = MyClass.new
v2 = MyClass.new
def v1.v1_method
  p "method of v1"
end
class << v1
  def v1_singleton_method
    p "v1_single_method"
  end
end

v1.v1_method
v2.v1_method ## undefined method
{% endhighlight %}
以上v1_method和v1_singleton_method就是对象v1的单件方法，而v2没有这两个方法。那什么是单件类？单件类是每一个Ruby对象都有的隐藏类。上例中的v1_method就是v1单件类的实例方法。在上例中，再加一点输出代码：
{% highlight ruby %}
p v1.singleton_methods
p v1
p v1.singleton_class
p v1.singleton_class.instance_methods(false)
p v1.class
p v1.singleton_class.superclass

output:
[:v1_method, :v1_singleton_method]
#<Class:0x007ff8ef856048>
#<Class:#<Class:0x007ff8ef856048>>
[:v1_method, :v1_singleton_method]
MyClass
MyClass
{% endhighlight %}
 “#&lt;Class:0x007ff8ef856048&gt;”表示对象v1，“#&lt;Class:#&lt;Class:0x007ff8ef856048&gt;&gt;”就是v1隐藏的单件类。每一个ruby对象都有两个类：1.本身的类；2.单件类。注意到最后一行的输出，v1的单件类与v1的类(MyClass)存在父子关系。

前面说过类也是对象，将上面的机制套用到类上（如下例），会发现：**类方法的本质就是它的单件方法**。Ruby用单件类的机制实现了纯的面向对象。
{% highlight ruby %}
class MyClass
  def self.class_method
    p "MyClass::class_method"
  end
end
MyClass.class_method
p MyClass.singleton_class.instance_methods(false) ## [:class_method] 
{% endhighlight %}
对象、类、单件类，它们之间的关系可以用一张图来说明：

<img src="http://o9m7jnwwp.bkt.clouddn.com/%E5%9B%BE%E7%89%871.png-style1?v=13">

obj是MyClass的实例对象，#表示相应的单件类。图中横线的箭头表示singleton_class,竖线和两条曲线的箭头表示superclass。注意到Module也是一个类，它的superclass是Object，这样就构成了一个环。以上就是Ruby的对象模型，理解了这个模型之后，就清楚了Ruby的方法查找过程：“向右一步，然后向上查找”。



## 参考文献
  - [1] https://www.ibm.com/developerworks/cn/linux/l-cn-closure/index.html#resources
  - [2] 《Ruby元编程》

