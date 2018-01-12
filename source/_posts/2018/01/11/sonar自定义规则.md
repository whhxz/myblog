---
title: sonar自定义规则
date: 2018-01-11 14:40:29
categories: ['sonar']
tags: ['sonar', '代码质量']
---

公司准备使用sonar作为代码质量管理，在代码中string不允许直接定义和环境相关的String，只能从配置文件中读取key，需要自己写一个sonar规则。
因为检测的是java文件，直接用java开发个自定义组件。
查看各种规则可以使用的语言：[Support of Custom Rules by Language](https://docs.sonarqube.org/display/DEV/Adding+Coding+Rules)

编写规则的时候官方有demo：[Sonar Java Custom Rules](https://github.com/SonarSource/sonar-custom-rules-examples/tree/master/java-custom-rules)
<!-- more -->
### 编写自定义规则
1、新建maven项目、修改pom.xml为官方demo中pom.xml文件，只保留自己项目相关的信息。
2、复制官方demo中文件MyJavaFileCheckRegistrar、MyJavaRulesDefinition、MyJavaRulesPlugin、RulesList四个文件到本地项目中
3、修改复制过来的文件修改错误信息、以及项目文件名
4、修改RulesList.getJavaChecks中其他demo中规则
5、新建自定义规则文件StringValBlacklistCheck.java

```java

import org.sonar.check.Priority;
import org.sonar.check.Rule;
import org.sonar.plugins.java.api.JavaFileScanner;
import org.sonar.plugins.java.api.JavaFileScannerContext;
import org.sonar.plugins.java.api.tree.BaseTreeVisitor;
import org.sonar.plugins.java.api.tree.LiteralTree;

import java.util.ArrayList;
import java.util.List;
import java.util.regex.Pattern;

import static org.sonar.plugins.java.api.tree.Tree.Kind.STRING_LITERAL;

/**
 * StringValBlacklistCheck
 * Created by whhxz on 2018/1/10.
 */
 //规则描述
@Rule(
        key = "StringValBlacklistCheck",
        name = "字符串值检查",
        description = "String不能直接设置与环境相关的值：如：dev1.fn、beta1.fn、idc1.fn、IP 等",
        priority = Priority.CRITICAL,
        tags = {"disable"})
public class StringValBlacklistCheck extends BaseTreeVisitor implements JavaFileScanner {
    private static List<Pattern> blacklist = new ArrayList<>();
    private JavaFileScannerContext context;
    //正则表达式匹配字符串规则
    static {
        //环境相关域名
        blacklist.add(Pattern.compile(".*(dev|beta|idc)[0-9.]*fn.*"));
        //ip4
        blacklist.add(Pattern.compile(".*\\d+\\.\\d+\\.\\d+\\.\\d+.*"));
    }

    @Override
    public void scanFile(JavaFileScannerContext context) {
        this.context = context;

        scan(context.getTree());

    }
    //重写父类方法
    @Override
    public void visitLiteral(LiteralTree tree) {
        //判断是否String
        if (tree.is(STRING_LITERAL)) {
            String strVal = tree.value();
            for (Pattern pattern : blacklist) {
                if (pattern.matcher(strVal).matches()) {
                    //不符合规则提示
                    context.reportIssue(this, tree, String.format("字符串：%s 不符合规则需要修改，字符串不允许直接出现与环境相关值", strVal));
                }
            }
        }
        super.visitLiteral(tree);
    }
}
```
6、在RulesList.getJavaChecks中添加定义的规则StringValBlacklistCheck。修改pom.xml中插件sonar-packaging-maven-plugin中pluginClass为之前MyJavaRulesPlugin修改后的名字
7、在resource建立包org.sonar.l10n.java.rules.squid
（命名规则参考：MyJavaRulesDefinition.RESOURCE_BASE_PATH定义的路径）
8、新建文件StringValBlacklistCheck_java.html（命名规则参考：MyJavaRulesDefinition.addHtmlDescription）
```html
<p>字符串值检查</p>
<h2>Noncompliant Code Example</h2>
<pre>
    String str = "http://xxxx.dev1.fn/xxxx/xxxx";// Noncompliant
    String str = "http://xxxx.beta1.fn/xxxx/xxxx";// Noncompliant
    String str = "http://xxxx.idc1.fn/xxxx/xxxx";// Noncompliant
    String str = "http://10.11.23.12/xxxx/xxxx";// Noncompliant
</pre>
<h2>Compliant Solution</h2>
<pre>
    //从配置文件中读取key，统一管理
</pre>
```
9、新建文件StringValBlacklistCheck_java.json（命名规则参考：MyJavaRulesDefinition.addMetadata）
```json
{
  "title": "String不能直接设置与环境相关的值：如：dev1.fn、beta1.fn、idc1.fn、IP 等",
  "type": "VULNERABILITY",
  "status": "ready",
  "remediation": {
    "func": "Constant\/Issue",
    "constantCost": "5min"
  },
  "tags": [
    "disable"
  ],
  "defaultSeverity": "CRITICAL"
}
#type可以参考RuleType
#status可以参考RuleStatus
```
10、打包项目放入 **$SONAR_HOME/extensions/plugins** 中，重启sonar
11、打开sonar web界面，配置插件。
11、测试项目，通过maven命令：*mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar* 生成报告，通过最后提示的访问最后打印的url即访问相关信息。（可以通过传递参数sonar.host.url指定sonar地址，默认为本地）

在测试项目的时候，相关参数可以配置到~/.m2/setting.xml中：如sonar.host.url等，参考：[Analyzing with SonarQube Scanner for Maven](https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner+for+Maven)

参考：[Writing Custom Java Rules](https://docs.sonarqube.org/display/PLUG/Writing+Custom+Java+Rules+101)
