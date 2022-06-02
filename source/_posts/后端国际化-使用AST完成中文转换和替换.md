---
title: '后端国际化(三): 使用AST完成中文转换和替换'
date: 2022-06-02 12:43:59
tags:
---

## 前言

在完成中文的提取和过滤之后，我们需要对中文节点进行转换、翻译、替换。

## 文案转换

文案转换通常有一下几个情况
### 普通文案
比如"保存成功"，直接替换成对应的code即可

### 拼接的文案

`"用户" + user.name + "不能为空"`，会做占位符转换, 将起变成`"用户{0}不能为空"`, 然后替换成对应的code

### 注解中的文案
比如`"用户不能为空"`，需要替换成成对应的`{code}`，code两边需要加入{}，以符合要求。

## 节点替换

**普通文案**

`"保存成功"`
替换为
`I18nUtils.getMessage("save_succeeded")`

**拼接文案**

`"用户名" + user.name + "不能为空"`
替换为
`I18nUtils.getMessage("user_name_not_empty")`

**注解中的文案**

`@NotNull(message = "'type'不能为空")`
替换为
`@NotNull(message = "{type_cannot_be_empty}")`

**字段中的字符串**

`private static String abc = "你好"`
替换为
```java
private String getAbc(){
  return I18nUtils.getMessage("hello")
}
```

**MessageFormat.format中的文案**

`MessageFormat.format("用户名{0}不能为空", user.name)`
替换为
`I18nUtils.getMessage("user_name_not_empty", new Object[]{ user.name })`


## 替换原理

所谓替换，就是解析相应的AST树并转为目标树

### 普通文案AST树构造

```groovy
MethodInvocation getI18nCall(List<ASTNode> args){
    def methodInvocation = ast.newMethodInvocation()
    methodInvocation.setExpression(ast.newSimpleName(Config.getI18nClass()))
    methodInvocation.name = ast.newSimpleName("getMessage")
    methodInvocation.arguments().addAll(args)
    methodInvocation
}
```

### 复杂文案节点构造

![](/images/D927AF24-B23C-4D30-9D97-BC56371474EC.png)

> 源AST树(MessageFormat.format(“{0}”, row))

![](/images/0ED40480-AA9C-4A78-B6DA-7E4401E64475.png)

> 目标AST树(I18nUtils.getMessage(“{0}”, new Object[]{ row }))

```groovy
void replaceMessageFormatWithI18n(MethodInvocation methodInvocation, StringLiteral stringLiteral){
  def args = []
  // 提取MessageFormat.format中的第一个中文
  StringLiteral code = ASTNode.copySubtree(methodInvocation.getAST(), stringLiteral)
  // 翻译并替换code
  translateKey(code)

  // 获取MessageFormat.format剩余的参数
  List<ASTNode> retainExp = methodInvocation.arguments().subList(1, methodInvocation.arguments().size())
      .collect({ ASTNode.copySubtree(methodInvocation.getAST(), (ASTNode)it) })

  args.add(code)
  def array = ast.newArrayCreation()
  args.add(array)
  // 构造new Object节点
  array.setType(ast.newArrayType(ast.newSimpleType(ast.newSimpleName("Object"))))
  def arrayInitializer = ast.newArrayInitializer()
  // 将剩余的参数全部加入到new Object节点
  arrayInitializer.expressions().addAll(retainExp)
  array.setInitializer(arrayInitializer)

  // 转换为I18nUtils.getMessage形式
  convertStringLiteral(args, methodInvocation)
}
```