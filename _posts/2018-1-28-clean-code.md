---
layout: post
title: 优雅的代码
category: notes
tags: Ruby 重构 编程语言
published: true
---

* 文章目录
{:toc}

刚开始工作的时候，自己的第一要务是把代码写正确，不出错，且努力弥补自己经验上的差距。渐渐熟练之后，开始感觉自己写的代码，看起来不那么顺眼。一份好的代码，应该是**清晰**、**简洁**、**易于扩展**的。

最近利用空余时间读了《优雅的Ruby》《重构》这两本书。虽然书中所用到的编程语言不同，但我觉得思想都是通用的，无非是语言特性所带来的技巧可能有差异。这篇文章，主要是对自己代码的一个反思和总结。

## 不好的代码

现在回过头去读三四个月之前的代码，很明显能够感觉到各种不舒适。比如：很长的函数，复杂的逻辑，重复等等。在《重构》的第三章中，列出了很多具体的坏味道，这些倒不一定是百分百的真理。我认为，如果一份代码，读起来感觉很费劲，那么它一定有能优化的地方。从我自身出发，下面列几个比较容易做到的点。

## 函数结构

{% highlight ruby %}
function(){
  输入参数的处理;
  功能实现;
  返回值;
}
{% endhighlight %}

一个基本的函数结构，应该包含三个部分：输入、返回、功能实现。输入部分的职责是处理参数，包括空值、异常值、类型转换等等。如果参数不满足条件，应该**及时返回**。如果条件比较复杂的时候，func2在开始的时候就返回，会显得更为清晰。这种方法也叫做卫语句(Guard Clauses)。
{% highlight ruby %}
def func1
  if x > 5
   x += 1
  end
  return x
end
def func2
  return x if x <= 5
   x += 1
  return x
end
{% endhighlight %}

如果输入处理好之后，后面的实现会容易很多。返回值部分要考虑对调用者友好。根据实际情况，考虑是否将异常抛出，还是自己处理。如果返回值较多，可以考虑用对象封装一下。

## 单一职责

实际的业务逻辑总是会十分复杂，但是人脑却是偏好于简单的东西。一个复杂的问题总会被拆解成一个一个小问题，逐步解决。写代码也是一样：

- 一个函数只干一件事情
- 一个类只有一个职责
- 一个继承体系只有一项职责

类的设计，这里先不谈。回过头去看之前写的代码，过长的函数，里面包含了很多复杂的逻辑。可以将里面的逻辑梳理一下，拆成若干个小的函数。即可以避免重复，代码逻辑也更清晰。

## 避免和减少条件语句

代码中的条件语句往往最影响代码的“美观”。合理利用一些技巧和模式，能够简化和消除条件语句。例如如下发送消息的代码：

{% highlight ruby %}
def send_msg
  if type == 'type1'
    query ids with condition1
  elsif type == 'type2'
    query ids with condition2
  end

  send_msg_with_ids(ids)
end
{% endhighlight %}

实际的业务中，每一种类型都是比较复杂的查询条件，整个函数会显得十分复杂。如果我们把查询条件封装一下:

{% highlight ruby %}
class ConditionOne
  def query_ids
  end
end
class ConditionTwo
  def query_ids
  end
end

def get_condition(type)
  case type
  when 'type1'
    return ConditionOne.new
  when 'type2'
    return ConditionTwo.new
  end
end

def send_msg
  condition = get_condition(type)
  ids = condition.query_ids
  send_msg_with_ids(ids)
end
{% endhighlight %}

这样条件语句变得简单，send_msg也变得十分简洁。

一份好的代码，应该是像一篇好文章一样，读起来十分流畅，神清气爽。

-----------------------

一转眼，半年过去了。对自己的现状打个及格分吧。工作之后，属于自己的时间真是非常少，上一篇文章还是9月2号。其实有很多想去总结和学习的东西，但是不知道从何处下手。我觉得自己总是不够专注，什么都懂一点，但什么都不深入。学生时代，这可能没什么大毛病，但企业或者说是市场更偏向于某一领域的专家。以后，我应该专注于自己工作中的问题，把手头已有的事情做到极致，深挖背后的东西，不想太多。
