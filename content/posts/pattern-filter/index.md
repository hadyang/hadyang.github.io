---
title: 拦截器模式
date: 2017-03-19
categories: 设计模式
type: post
tags:
  - Java
---

拦截器模式在 Web 应用的 Fitler 中有应用，对于流式的数据流处理很有用。比如对于 Shell 命令的管道可以采用拦截器模式，提供一个命令的接口，并使用一个 Chain 来进行管理。

<!--more-->

拦截器的基本类结构用 UML 类图的方式进行表示：

![](pattern-filter.jpg)

其中 Filter 提供一个统一的接口， Chain 用来管理所有的 Filter，并提供一个`doNext()`方法来进行下一个拦截器的执行。

下面给出一个拦截器模式的示例代码：

```java
public abstract class Command {
    protected List<Character> options;

    protected List<String> params;

    public Command() {
        options = Lists.newArrayList();
        params = Lists.newArrayList();
    }

    public List<Character> getOptions() {
        return options;
    }

    public void setOptions(List<Character> options) {
        this.options = options;
    }

    public List<String> getParams() {
        return params;
    }

    public void setParams(List<String> params) {
        this.params = params;
    }

    public abstract String execute(CommandChain chain, String data);

}
```

```java
public class CommandChain {
    private List<Command> commandList;

    private int index;

    public CommandChain(List<Command> commandList) {
        this.commandList = commandList;
    }

    public String doNext(String data) {
        if (commandList.size() <= index) {
            return data;
        } else {
            return commandList.get(index++).execute(this, data);
        }
    }
}
```
