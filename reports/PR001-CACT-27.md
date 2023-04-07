[TOC]

# 编译原理研讨课实验PR001实验报告

## 任务说明

本次实验主要有以下两个任务：

* **任务一：熟悉`Antlr`的安装和使用**

  在此任务中，我们需要完成两个子任务：

  * 掌握如何安装`Antlr`
  * 了解如何编写文法文件并生成`lexer`和`parser`

* **任务二：完成词法分析和语法分析**

  在此任务中，我们需要完成两个子任务：

  * 编写`.g4`文件：根据给出的CACT文法规范，编写.g4文件，并通过`Antlr`生成`lexer`和`parser`，对输入的CACT语言源代码(`.cact`文件)进行词法和语法分析
  *  修改`Antlr`中默认的文法错误处理机制，使得在遇到词法和语法错误时，能进行相应的处理

## 成员组成

夏瑞阳、陈兢宇、余秋初

## 实验设计

### 设计思路

####  任务一设计思路

对于任务一，课程提供的`PR001-CACT-实验说明.pdf`里已经做了详细说明，我们按图索骥做下来即可完成，因此在报告中不再赘述。

这里可能需要注意的是我们在编写`.g4`文件需要遵守的一些语法规范，例如：

* 文件开头`grammar grammar_name`，这里的`grammar_name`需要与文件名保持一致
* 对于一条`parser rule`，其标识符应以小写字母开头
* 对于一条`lexer rule`，其标识符应以大写字母开头
* ...

#### 任务二设计思路

对于任务二，我们对两个子任务的设计思路分别进行阐述。

##### .g4文件的设计思路

.g4文件的编写主要依靠CACT语言规范。根据语言规范中定义的文法，编写.g4文件。

* 对于Parser部分，.g4文件只需要根据语言规范以及所给出的例子，比葫芦画瓢，照着编写即可。
* 对于Lexer部分，除了规范中给出的CACT语言文法定义之外，我们根据自己的需求自行设计标识符`Ident`，整型常量`IntConst`，布尔型常量`BoolConst`，单精度浮点常量`FloatConst`，双精度浮点常量`DoubleConst`。

##### 错误处理机制的设计思路

经过查阅资料，对实现修改错误处理机制有三种可能的思路：

* 编写`DefaultErrorStrategy`类的子类，并且覆盖其`recover`, `sync`, `recoverInline`三个方法，我们可以将这三个方法用空方法进行覆盖，使得语法分析器在遇到第一个错误处即退出，即采用`BailErrorStrategy`

  ```C++
  class BailErrorStrategy :: public DefaultErrorStrategy
  {
      void recover(Parser recongizer, RecongnitionException e)
      {
          ;
      }
      
      void recoverInline(Parser recongnizer)
      {
          ;
      }
      
      void sync(Parser recongizer)
      {
          ;
      }
  };
  ```

  除此之外，我们还需要创建一个新的`BailErrorStrategy`实例，并且令语法分析器使用它来替代默认的错误处理策略

  ```c++
  parser.setErrorHandler(new BailErrorStrategy());
  ```

  我们还应当令词法分析器遇到第一个词法错误处退出，需要覆盖`Lexer`类中的`recover`方法。

* 使用`visitor`模式遍历语法分析树。`visitor`模式可以指定方法的返回值，同时，在`BaseVisitor`中提供了`visitErrorNode`方法，该方法将在遍历到错误节点时被调用。基于此，我们可以让遍历器在遇到出错节点时返回非零值1，最后在`main.cpp`里，通过

  ```C++
  int result = ErrorVisitor.visit(tree);
  ```

  进行返回值传递，并且返回相应`result`

* 使用`listener`模式遍历语法树。实现方法同`visitor`模式基本一致，但由于其不能指定返回值，因此考虑在遍历到出错节点时直接执行`exit(1)`

### 实验实现

##### .g4文件编写的实现

对于自行设计实现的各种常量终结符：

* 标识符`Ident`,整型常量`IntConst`,布尔型常量`BoolConst`与给出的例子保持一致即可。

* 单精度浮点常量`FloatConst`：

  ```
  FloatConst
      : DoubleConst ('f' | 'F') ?
      ;
  ```

  在浮点数后面加上f或者是F用于区分单精度浮点数和双精度浮点数，在数值的表示上单精度浮点数与双精度浮点数的表示方法一致。

* 双精度浮点常量`DoubleConst`：

  ```
  DoubleConst
      : DoubleBase Exponent ? 
      ;
  fragment
  DoubleBase
      : Digit+ '.' Digit*
      | '.' Digit+
      | NonzeroDigit Digit*
      ;
  fragment
  Exponent
      :('E' | 'e') ('+' | '-') ? NonzeroDigit Digit*
      ;
  ```

​		对于双精度浮点数，将其拆分为两个部分，一个是数值部分`DoubleBase`，另一个是指数部分`Exponent`，其中指数部分可有可无，例如1.22和1.22e-3。对于`DoubelBase`，一般情况是`Digit+ '.' Digit*`，例如1.22；CACT中也可以省略整数部分`'.' Digit+`，例如.22e-3；或者是没有小数部分`NonzeroDigit Digit*`，例如3e-2。对于指数部分在后面添加指数次方即可，如E-55，e-55，E+4，E4等。

##### 错误处理机制的实现

经过和老师交流后，由于实验的关注点在语法分析和语法规范本身，我们选择了最简洁的使用`listener`模式遍历语法树的做法。具体实现的代码如下：

```C++
void SemanticAnalysis::visitErrorNode(antlr4::tree::ErrorNode * e)
{
    exit(1);//遍历到错误节点直接退出
    //std::cout << "\tError: " << e->getText().c_str() << std::endl;
}
```

这里的`SemanticAnalysis`继承自`CACTBaseListener`，我们覆盖了它的`visitErrorNode`方法，使得遍历到错误节点后程序会直接退出。


## 总结

### 实验结果总结

![](https://raw.githubusercontent.com/0x1z2c3v/image/master/image-20220410111247606.png)
<center style="font-size:14px;color:#000;text-decoration:underline">图 1. ./PR001_test.sh执行结果</center> 

如图所示，我们的程序能够对提供的27个测试样例给出正确的分析结果。（输出的错误提示信息来自`Antlr`）



### 分成员总结

#### 余秋初的总结

在本次实验中，我通过调试.g4文件和修改`Antlr`错误处理机制，了解了`Antl4`的语法的基本规范和作用机制，对于如何利用`Antlr`进一步进行语义分析也有了一些大概的思路。调试.g4文件的过程也让我对编译原理理论课上的词法分析和语法分析过程有了更加直观和深刻的理解。

#### 陈兢宇的总结

通过本次实验，简单了解了`Antlr`的工作流程，学习简单的正则表达式的书写规范。通过分析错误处理流程，对词法分析和语法分析，以及抽象语法树有了进一步的理解。

#### 夏瑞阳的总结

在本次实验中，我通过编写.g4文件，了解了`Antl4`的语法的基本规范和作用机制，例如`fragment`关键字和`Listener`、`Visitor`模式等。我对之前理论课中讲到的词法分析、语法分析有了更深的回顾和认识，对之后理论课中讲到的语法制导翻译以及基于属性的语义分析有了更多的理解。


