---
title: Java设计模式-解释器模式（Interpreter Pattern）
date: 2017-10-27 13:52:36
categories: ['设计模式']
tags: ['设计模式','解释器模式','Interpreter Pattern']
---

### 解释器模式
> 定义一个语言的文法，并且建立一个解释器来解释该语言中的句子，这里的“语言”是指使用规定格式和语法的代码。解释器模式是一种类行为型模式。

解释器模式是一种使用频率相对较低但学习难度较大的设计模式，它用于描述如何使用面向对象语言构成一个简单的语言解释器。在某些情况下，为了更好地描述某一些特定类型的问题，我们可以创建一种新的语言，这种语言拥有自己的表达式和结构，即文法规则，这些问题的实例将对应为该语言中的句子。此时，可以使用解释器模式来设计这种新的语言。对解释器模式的学习能够加深我们对面向对象思想的理解，并且掌握编程语言中文法规则的解释过程。

由于表达式可分为终结符表达式和非终结符表达式，因此解释器模式的结构与组合模式的结构有些类似，但在解释器模式中包含更多的组成元素。
<!-- more -->
UML类图如下：
![](http://otxnth5wx.bkt.clouddn.com/20171027屏幕快照2017-10-27下午2.07.29.png)
![](http://otxnth5wx.bkt.clouddn.com/20171027屏幕快照2017-10-27下午2.07.45.png)
```java
/**
 * 抽象表达式
 */
abstract class AbstractExpression {
    public abstract void interpret(Context ctx);
}

/**
 * 终结符表达式
 */
class TerminalExpression extends  AbstractExpression {
    public  void interpret(Context ctx) {
        //终结符表达式的解释操作
    }
}

/**
 * 非终结符表达式
 * 非终结符表达式中可以包含终结符表达式
 */
class NonterminalExpression extends  AbstractExpression {
    private  AbstractExpression left;
    private  AbstractExpression right;

    public  NonterminalExpression(AbstractExpression left,AbstractExpression right) {
        this.left=left;
        this.right=right;
    }

    public void interpret(Context ctx) {
        //递归调用每一个组成部分的interpret()方法
        //在递归调用时指定组成部分的连接方式，即非终结符的功能
    }
}

/**
 * 存储解释器之外的一些全局信息
 */
class Context {
    private Map<String, String> map = new HashMap<>();

    public void assign(String key, String value) {
        //往环境类中设值
    }

    public String lookup(String key) {
        //获取存储在环境类中的值
        return map.get(key);
    }
}
```
* AbstractExpression（抽象表达式）：在抽象表达式中声明了抽象的解释操作，它是所有终结符表达式和非终结符表达式的公共父类。
* TerminalExpression（终结符表达式）：终结符表达式是抽象表达式的子类，它实现了与文法中的终结符相关联的解释操作，在句子中的每一个终结符都是该类的一个实例。通常在一个解释器模式中只有少数几个终结符表达式类，它们的实例可以通过非终结符表达式组成较为复杂的句子。
* NonterminalExpression（非终结符表达式）：非终结符表达式也是抽象表达式的子类，它实现了文法中非终结符的解释操作，由于在非终结符表达式中可以包含终结符表达式，也可以继续包含非终结符表达式，因此其解释操作一般通过递归的方式来完成。
* Context（环境类）：环境类又称为上下文类，它用于存储解释器之外的一些全局信息，通常它临时存储了需要解释的语句。

### 数学表达式计算
在Js中通过eval("1+2+3+4")可以得出字符串数学表达式的值，在java中是无法直接解析这种字符串，通过解释器设计模式，设计可以解决计算字符串计算。
如：字符串"1+2*3-4"
使用形式化语言表示为：
```
expression ::= value | operation
operation ::= expression '+' expression | expression '-'  expression | expression '*'  expression | expression '/'  expression
value ::= an integer
```
使用抽象语法树表示为如下：
![](http://otxnth5wx.bkt.clouddn.com/20171027屏幕快照2017-10-27下午3.31.56.png)

设计四则运算：
1、中缀表达式转后缀表达式
1.1、从左到右遍历中缀表达式的每一个数字和运算符
1.2、如果数字就输出（即存入后缀表达式）
1.3、如果是右括号，则弹出左括号之前的运算符
1.4、如果优先级低于栈顶运算符，则弹出栈顶运算符，并将当前运算符进栈。
1.5、遍历结束后，将栈则剩余运算符弹出。
2、后缀表达式计算
2.1、从左到右遍历后缀表达式，遇到数字就进栈，遇到符号，就将栈顶的两个数字出栈运算，运算结果进栈，直到获得最终结果。

UML类图如下：
![](http://otxnth5wx.bkt.clouddn.com/20171027屏幕快照2017-10-27下午6.09.30.png)
```java
/**
 * 表达式抽象类
 */
abstract class AbstractExpression {
    public abstract int interpret();
}

/**
 * 数字表达式
 * 终结符表达式
 */
class NumberExpression extends AbstractExpression {
    private int num;

    public NumberExpression(int num) {
        this.num = num;
    }

    public NumberExpression(String num) {
        this.num = Integer.valueOf(num);
    }

    @Override
    public int interpret() {
        return num;
    }
}

/**
 * 抽象运算符表达式
 * 非终结符表达式
 */
abstract class AbstractOperatorExpression extends AbstractExpression {
    // 每个运算符都有左右两个参数进行运算，因此抽象到父类中
    protected AbstractExpression left;
    protected AbstractExpression right;

    public AbstractOperatorExpression(AbstractExpression left, AbstractExpression right) {
        this.left = left;
        this.right = right;
    }
}

/**
 * 加法运算符
 */
class AddExpression extends AbstractOperatorExpression {

    public AddExpression(AbstractExpression left, AbstractExpression right) {
        super(left, right);
    }

    @Override
    public int interpret() {
        return left.interpret() + right.interpret();
    }
}

/**
 * 减法运算符表达式
 */
class MinusExpression extends AbstractOperatorExpression {

    public MinusExpression(AbstractExpression left, AbstractExpression right) {
        super(left, right);
    }

    @Override
    public int interpret() {
        return left.interpret() - right.interpret();
    }
}

/**
 * 乘法运算符
 */
class MultiplyExpression extends AbstractOperatorExpression {

    public MultiplyExpression(AbstractExpression left, AbstractExpression right) {
        super(left, right);
    }

    @Override
    public int interpret() {
        return left.interpret() * right.interpret();
    }
}

/**
 * 除法运算符
 */
class DivideExpression extends AbstractOperatorExpression {

    public DivideExpression(AbstractExpression left, AbstractExpression right) {
        super(left, right);
    }

    @Override
    public int interpret() {
        return left.interpret() / right.interpret();
    }
}

/**
 * 存储全局变量
 */
class Context {

    private static Map<String, Integer> priority = new HashMap<>(8);

    private static Set<String> operator = new HashSet<>();

    static {
        operator.add("+");
        operator.add("-");
        operator.add("*");
        operator.add("/");

        //运用运算符ASCII码-40做索引的运算符优先级
        priority.put("(", 0);
        priority.put(")", 3);
        priority.put("*", 2);
        priority.put("+", 1);
        priority.put(",", -1);
        priority.put("-", 1);
        priority.put(".", 0);
        priority.put("/", 2);

    }

    private AbstractExpression expression;

    /**
     * 解析
     *
     * @param value
     */
    public void analyse(String value) {
        Stack<AbstractExpression> stack = new Stack<>();

        String suffix = inffixToSuffix(value);
        for (String str : suffix.split(" ")) {
            //运算符入栈
            if (!priority.containsKey(str)){
                AbstractExpression expression = new NumberExpression(str);
                stack.push(expression);
            } else {
                AbstractExpression rightExpression = stack.pop();
                AbstractExpression leftExpression = stack.pop();

                AbstractOperatorExpression abstractExpression;
                switch (str){
                    case "+":
                        abstractExpression = new AddExpression(leftExpression, rightExpression);
                        break;
                    case "-":
                        abstractExpression = new MinusExpression(leftExpression, rightExpression);
                        break;
                    case "*":
                        abstractExpression = new MultiplyExpression(leftExpression, rightExpression);
                        break;
                    case "/":
                        abstractExpression = new DivideExpression(leftExpression, rightExpression);
                        break;
                    default:
                        throw new RuntimeException("参数异常");
                }

                int num = abstractExpression.interpret();
                NumberExpression numberExpression = new NumberExpression(num);
                stack.push(numberExpression);
            }
        }
        expression = stack.pop();
    }

    //计算结果
    public int calculate() {
        return expression.interpret();
    }

    /**
     * 将中缀表达式转换成后缀表达式
     * @param expressionStr
     * @return
     */
    public static String inffixToSuffix(String expressionStr) {
        Stack<String> stack = new Stack<>();
        //放入最小优先级的符号
        stack.push(",");
        StringBuilder inffix = new StringBuilder(expressionStr.replaceAll(" ", ""));
        StringBuilder suffix = new StringBuilder();
        String element;
        while (inffix.length() > 0) {
            element = nextEle(inffix);
            //判断是否是运算符
            if (!priority.containsKey(element)) {
                suffix.append(element).append(" ");

            } else if (")".equals(element)) {//如果是反括号，则从栈中弹出数据直到弹出值为正括号
                String tmp = stack.pop();
                while (!"(".equals(tmp)){
                    suffix.append(tmp).append(" ");
                    tmp = stack.pop();
                }
            } else if ("(".equals(element)
                    || priority.get(element) >= priority.get(stack.peek())) {//如果是正括号或者当前符号优先级大于等于栈顶的优先级，那么放入栈中
                stack.push(element);
            } else {//当前优先级小于栈顶优先级，取出栈顶
                String tmp = stack.peek();
                while (priority.get(element) < priority.get(tmp)){
                    tmp = stack.pop();
                    suffix.append(tmp).append(" ");
                    tmp = stack.peek();
                }
                suffix.append(element).append(" ");
            }
        }
        while (stack.size() > 0 && !",".equals(stack.peek())){
            suffix.append(stack.pop()).append(" ");
        }

        return suffix.toString();
    }

    /**
     * 获取下一个节点
     *
     * @param expressionStr
     * @return
     */
    private static String nextEle(StringBuilder expressionStr) {
        StringBuilder nextStr = new StringBuilder();
        char c = expressionStr.charAt(0);
        expressionStr.deleteCharAt(0);
        nextStr.append(c);
        while (isNum(c) && expressionStr.length() > 0 && isNum(c = expressionStr.charAt(0))) {
            expressionStr.deleteCharAt(0);
            nextStr.append(c);
        }
        return nextStr.toString();
    }

    private static boolean isNum(char c) {
        return (c >= '0' && c <= '9');
    }
}

public class Main {
    public static void main(String[] args) throws InterruptedException, ScriptException {
        String str = "8 + (9-1)*8/2+8*2";
        Context context = new Context();
        context.analyse(str);
        System.out.println(context.calculate());
        //采用引擎执行脚本
        ScriptEngineManager scriptEngineManager = new ScriptEngineManager();
        ScriptEngine nashorn = scriptEngineManager.getEngineByName("nashorn");
        Object jsReturn = nashorn.eval(str);
        System.out.println(jsReturn);
    }
}
```
* 参考：http://blog.csdn.net/kinglearnjava/article/details/48786829

### 解释器模式总结
在平常使用java.util.Pattern、java.text.Format等都是采用了解释器设计模式。

解释器模式为自定义语言的设计和实现提供了一种解决方案，它用于定义一组文法规则并通过这组文法规则来解释语言中的句子。虽然解释器模式的使用频率不是特别高，但是它在正则表达式、XML文档解释等领域还是得到了广泛使用。与解释器模式类似，目前还诞生了很多基于抽象语法树的源代码处理工具，例如Eclipse中的Eclipse AST，它可以用于表示Java语言的语法结构，用户可以通过扩展其功能，创建自己的文法规则。

在实际项目中，非特殊情况下可以不适用解释器模式，采用脚本语言如JS、Python等。

**应用场景：**
* 需要解释执行的语言中的句子表示为一个抽象语法树。
* 一些重复出现的问题可以用一种简单的语言来进行表达。
* 执行效率不是关键问题。【注：高效的解释器通常不是通过直接解释抽象语法树来实现的，而是需要将它们转换成其他形式，使用解释器模式的执行效率并不高。】

**优点：**
* 易于改变和扩展文法。由于在解释器模式中使用类来表示语言的文法规则，因此可以通过继承等机制来改变或扩展文法。
* 每一条文法规则都可以表示为一个类，因此可以方便地实现一个简单的语言。
* 实现文法较为容易。在抽象语法树中每一个表达式节点类的实现方式都是相似的，这些类的代码编写都不会特别复杂，还可以通过一些工具自动生成节点类代码。

**缺点：**
* 对于复杂文法难以维护。在解释器模式中，每一条规则至少需要定义一个类，因此如果一个语言包含太多文法规则，类的个数将会急剧增加，导致系统难以管理和维护，此时可以考虑使用语法分析程序等方式来取代解释器模式。
* 执行效率较低。由于在解释器模式中使用了大量的循环和递归调用，因此在解释较为复杂的句子时其速度很慢，而且代码的调试过程也比较麻烦。
