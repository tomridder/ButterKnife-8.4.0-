---
title: ButterKnife源码解析（三）
date: 2022-02-03 19:54:14
tags:
---
# ButterKnife 8.4.0 源码分析（三）

3.获取 BindingClass，有缓存机制, 没有则创建

这里先介绍下这个targetClass变量 

```java
Map<TypeElement, BindingClass> targetClassMap = new LinkedHashMap<>();
```

这个targetClassMap 就是一个Map。键enclosingElement则是@BindView注解所在的类，（这里非常重要，后面会说目的）而值BindingClass则是包含了@BindView注解修饰的元素以及 @BindView的值的一个类。

这里我梳理了下targetClassMap里面成员变量的关系，画成了一张图。

![](BK类图.png)

我们这里先看parseBindView 函数中45行的 getId(id)函数 ，点击去会看到

```java
  private Id getId(int id) {
    if (symbols.get(id) == null) {
      symbols.put(id, new Id(id));
    }
    return symbols.get(id);
  }
```

getId(int id)得到的是Id对象，看名称我们也知道这个Id类存储的是id及相关的信息（就是我们第二步得到的id值）

至于BindingClass具体的我们可以点BindingClass进去看看。

```java
final class BindingClass {
//....
  private final Map<Id, ViewBindings> viewIdMap = new LinkedHashMap<>();
//....
}
```

他的内部也有一个以Id，ViewBinds为键值对的Map。我们点击去看ViewBindings 其实这里主要发现其有一个FieldViewBinding的对象和 set方法。

```java
final class ViewBindings {

    //.....
    private FieldViewBinding fieldBinding;
    
      public void setFieldBinding(FieldViewBinding fieldBinding) {
    if (this.fieldBinding != null) {
      throw new AssertionError();
    }
    this.fieldBinding = fieldBinding;
    
    //.....
  }
」
```

至于FieldViewBinding的内部我们可以看到主要就是以下三个变量。

```java
final class FieldViewBinding implements ViewBinding {
  //@BindView(R.id.tvWord)
  // TextView word;  
  //name就是word
  private final String name;
  //类型的名字 这里则是 TextView
  private final TypeName type;
  //是否要求可为空
  private final boolean required;

  FieldViewBinding(String name, TypeName type, boolean required) {
    this.name = name;
    this.type = type;
    this.required = required;
  }

  public String getName() {
    return name;
  }

  public TypeName getType() {
    return type;
  }

  public ClassName getRawType() {
    if (type instanceof ParameterizedTypeName) {
      return ((ParameterizedTypeName) type).rawType;
    }
    return (ClassName) type;
  }

  @Override public String getDescription() {
    return "field '" + name + "'";
  }

  public boolean isRequired() {
    return required;
  }
}
```

把这些细节了解完了 我们可以来开始看第三步

```java
   BindingClass bindingClass = targetClassMap.get(enclosingElement);
    if (bindingClass != null) {
      ViewBindings viewBindings = bindingClass.getViewBinding(getId(id));
      if (viewBindings != null && viewBindings.getFieldBinding() != null) {
        FieldViewBinding existingBinding = viewBindings.getFieldBinding();
        // id 已经被绑定过
        error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
            BindView.class.getSimpleName(), id, existingBinding.getName(),
            enclosingElement.getQualifiedName(), element.getSimpleName());
        return;
      }
    } else {
      bindingClass = getOrCreateTargetClass(targetClassMap, enclosingElement);
    }
```

1.如果不为空，则根据id获取保存在bindingClass中的ViewBindings实例， 
如果viewBindings不为空且viewBindings.getFieldBinding（得到的就是内部的FieldViewBinding 对象）不为空则抛出异常，什么意思呢？就是说一个id不能bind多次。 
2.如果为空，则bindingClass = getOrCreateTargetClass(targetClassMap, enclosingElement);

获取并返回，参数是最初的那个map和父节点。

```java
  private BindingClass getOrCreateTargetClass(Map<TypeElement, BindingClass> targetClassMap,
      TypeElement enclosingElement) {
    BindingClass bindingClass = targetClassMap.get(enclosingElement);
    //再次判断
    if (bindingClass == null) {
     //获取父节点的类型名字，这个不用太关心
      TypeName targetType = TypeName.get(enclosingElement.asType());
      if (targetType instanceof ParameterizedTypeName) {
        targetType = ((ParameterizedTypeName) targetType).rawType;
      }

      //获取该enclosingElement就是父节点所在的包名称

      String packageName = getPackageName(enclosingElement);
      //类名字
      String className = getClassName(enclosingElement, packageName);
      //根据包名称和类名称获取bindingClassName实体
      //并且加入了_ViewBinding 哈哈，有点意思是了。不是吗？？
      ClassName bindingClassName = ClassName.get(packageName, className + "_ViewBinding");

      //是否是final 类  后面利用javapoet生成java类的时候会用到
      boolean isFinal = enclosingElement.getModifiers().contains(Modifier.FINAL);

      //创建了一个BindingClass实例
      bindingClass = new BindingClass(targetType, bindingClassName, isFinal);
      //加入集合，缓存
      targetClassMap.put(enclosingElement, bindingClass);
    }
    return bindingClass;
  }
```

到此我们的parseBindView的步骤3就完了。 

4.生成FieldViewBinding 实体

```java
    //@BindView(R.id.tvWord)
    // TextView word;  
    //name就是word
    String name = element.getSimpleName().toString();
     //类型的名字 就是TextView
    TypeName type = TypeName.get(elementType);
    //是否要求可为空
    boolean required = isFieldRequired(element);
     //生成FieldViewBinding实体
    FieldViewBinding binding = new FieldViewBinding(name, type, required);
```

5.

```java
//加到bindingClass中的成员变量的实体的集合中，方便生成java源文件也就是xxxxx_ViewBinding文件的成员变量的初始化存在。（其实这里也是通过Id先得到ViewBindings，再把FieldViewBinding加入到bindingClass的ViewBindings中 点击去看一遍可以知道）
bindingClass.addField(getId(id), binding);
// @BindView所在的类所组成的set，因为不能有重复的类，所以用set。
erasedTargetNames.add(enclosingElement);
```

相信讲到这里你已经对Bindingclass这个类是什么，有一个大致的了解了。（它包含了@BindView注解所修饰的元素名，元素种类，元素是否可以为空，@BindView的id值，@BindView注解所修饰的元素所在的类的类型，以及这个类名对应的Activity_ViewBinding类名，@BindView注解所修饰的元素所在的类是否为final类 这几项信息）

现在我们再回过头来 findAndParseTargets()中对于 @BindView 注解的处理

```java
 private Map<TypeElement, BindingClass> findAndParseTargets(RoundEnvironment env) {
    Map<TypeElement, BindingClass> targetClassMap = new LinkedHashMap<>();
    Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();
     //...
      // Process each @BindView element.
    for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
      // we don't SuperficialValidation.validateElement(element)
      // so that an unresolved View type can be generated by later processing rounds
      try {
        parseBindView(element, targetClassMap, erasedTargetNames);
      } catch (Exception e) {
        logParsingError(element, BindView.class, e);
      }
    }
     //...
     return targetClassMap;
 }
```

其实ButterKnife做的处理就是把所有被@BindView注解过的元素做一个规整，前面说的以enclosingElement为键的原因就是把在同一个类中的被@BindView注解过的元素放到一起，便于下一步直接遍历targetClassMap用javapoet生成一个有规律的java源文件。画个图阐述下我的意思。
![](BK转换图.png)

ok 这章节就结束了，接下来我们看下jake大神是如何将targetClassMap转换成java文件的。