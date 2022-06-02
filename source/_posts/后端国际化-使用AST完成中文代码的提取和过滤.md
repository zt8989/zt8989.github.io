---
title: '后端国际化(二): 使用AST完成中文代码的提取和过滤'
date: 2022-06-02 12:43:33
tags:
---

## 提取中文

本工具使用org.eclipse.jdt.core将java代码转换成AST

1. 将java代码转换成AST树
```groovy
def parser = new Parser()
CompilationUnit result = parser.ast(content)
Document document = new Document(src.text)
def ast = result.getAST()
```

2. 使用Vistior模式遍历所有字符串
```groovy
List<StringLiteral> visitChineseText(CompilationUnit result){
    var translateList = []
    result.accept(new ASTVisitor(){
        @Override
        boolean visit(StringLiteral node) {
            if(node.toString() =~ Config.DOUBLE_BYTE_REGEX){
                translateList.add(node)
            }
            return super.visit(node)
        }
    })
    return translateList
}
```


## 过滤

### 原理：熟悉AST

> 以下是`logger.info("异常信息"+e)`的AST


![logger.info("异常信息"+e)](/images/A4C441DD-29F6-478F-A0BB-361572EA6D9B.png)

如果我们需要过滤`logger.info("异常信息"+e)`，我们需要lookup到MethodInvocation节点，然后查看MethodInvocation的expression是否为SimpleName类型，且getIdentifier() == "logger"

### 需要过滤的中文

kiwi中实现的过滤器支持过滤以下中文：

* 去除注释
* 过滤log中的中文
* 系统配置的中文
* 参与业务逻辑的中文比如 "中文".equsals("中文")
* 正则匹配的中文字符（只包含标点）
* 存在于注解中的中文
* 存在于枚举的中文
* 注释中添加了kiwi-disable-method

### 过滤器实现

> 所有的过滤器实现了`Predicate<StringLiteral>`接口，用来判断中文是否过滤

常用的过滤器有：

1. 注解过滤器

```groovy
boolean isInAnnotation(StringLiteral stringLiteral){
    def parent = stringLiteral.parent
    // 以StringLiteral节点向上查找，如果找到了注解，则返回true
    while (parent != null && !(parent instanceof TypeDeclaration)) {
        if(parent instanceof Annotation){
            def memberValuePair = AstUtils.lookupUntilASTNode(stringLiteral, MemberValuePair.class)
            if(memberValuePair.isPresent()){
                return !shouldExclude(memberValuePair.get(), parent)
            }
            return true
        }
        parent = parent.parent
    }
    return false
}
```

2. 常量过滤器
3. 枚举过滤器

> 通常Enum里的中文无需翻译，也无法替换，因为不能在初始化的时候调用方法

```java
public enum RpFlagEnum {
    /**
     * 应付
     */
    P("P","应付"),

    /**
     * 应收
     */
    R("R","应收");
}
```
```groovy
  def optMethod = AstUtils.lookupUntilASTNode(stringLiteral, EnumConstantDeclaration.class)
  if (optMethod.isPresent()) {
      return true
  }
  return false
```

4. 日志过滤

> 过滤 log.info("中文")

```groovy
        def optMethod = AstUtils.lookupUntilASTNode(stringLiteral, MethodInvocation.class)
        if(optMethod.isPresent()){
            def methodName = optMethod.get().getExpression()
            // log.info(xxx)
            if(methodName instanceof SimpleName){
                def simpleName = (SimpleName)methodName
                if(isLogInfo(simpleName.getIdentifier())){
                    return true
                }
                // this.log.info(xxx)
            } else if(methodName instanceof  FieldAccess){
                def fieldAccess = (FieldAccess)methodName
                def fieldName = fieldAccess.getName()
                if(isLogInfo(fieldName.getIdentifier())){
                    return true
                }
            }
        }
        return false
```

5. Main方法过滤
6. I18n方法过滤
7. 注释中添加了kiwi-disable-method

> 某些情况下我们需要过滤的中文属于业务逻辑，但是代码无法判断，需要添加注释跳过

```java
/**
*
* kiwi-disable-method
*/
pubic String abc(){
  return "中文";
}
```

```groovy
    boolean hasMethodComment(StringLiteral stringLiteral) {
        def parent = stringLiteral.parent
        while (parent != null && !(parent instanceof TypeDeclaration)) {
            if(parent instanceof MethodDeclaration){
                def doc = parent.getJavadoc()
                return doc.toString().contains(methodComment)
            }
            parent = parent.parent
        }
        return false
    }
```

8. 正则过滤器(正则匹配的中文字符（只包含标点）)
9. 参与业务逻辑的中文比如 "中文".equsals("中文")

```groovy
   boolean hasStringEquals(StringLiteral stringLiteral){
        def optMethod = AstUtils.lookupUntilASTNode(stringLiteral, MethodInvocation.class)
        if(optMethod.isPresent()){
            def method = optMethod.get()
            if(method.name.identifier == "equals"){
                return true
            }
        }
        return false
    }
```