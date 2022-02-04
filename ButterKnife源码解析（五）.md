---
title: ButterKnife源码解析（五）
date: 2022-02-03 20:07:15
tags:
---
# ButterKnife 8.4.0 源码分析（五）

## 自己实现ButterKnife的@BindView方法

### 1.项目目录

注意这里new Module时需要生成一个 java的module 而不是Android的module，否则某些特定方法会得不到。

![](BK目录图.png)

### 2.java包 annotation部分

首先我们来看 我们自定义的注解 @BindView部分

```java
/**
 * 定义修饰的元素为 变量
 * 定义保留到 编译期
 * 有一个整形的int类型的值
 */
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.CLASS)
public @interface BindView {
     int value();
}
```

接下来是我们Butterknife对外提供的接口

```java
package com.yunxiao.dev.annotation;


import java.lang.reflect.Constructor;

public class ButterKnife {

    public static void bind(Object activity) {
        Class<?> aClass = activity.getClass();
        // getName() 带包名
        String bindName = aClass.getName() + "$$ViewBinder";
        try {
            Class<?> bindClazz =Class.forName(bindName);
            //首先获取到这个类有参数的构造方法
            Constructor<?> constructor = bindClazz.getConstructor(activity.getClass());
            // 通过反射生成实例 调用构造方法
            constructor.newInstance(activity);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 3.自定义处理器 AnnotationCompiler

```java
package com.yunxiao.dev.annotaion_compiler;

import com.google.auto.service.AutoService;
import com.yunxiao.dev.annotation.BindView;

import java.io.IOException;
import java.io.Writer;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import javax.annotation.processing.AbstractProcessor;
import javax.annotation.processing.Filer;
import javax.annotation.processing.ProcessingEnvironment;
import javax.annotation.processing.Processor;
import javax.annotation.processing.RoundEnvironment;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.Element;
import javax.lang.model.element.PackageElement;
import javax.lang.model.element.TypeElement;
import javax.lang.model.element.VariableElement;
import javax.lang.model.type.TypeMirror;
import javax.tools.JavaFileObject;

/**
 * 目的：生成一个java文件 去生成执行窗体的findVIewById的事件代码
 * <p>
 * 注解处理器
 * 1.继承
 * 2。注册服务 注册注解处理器
 */
@AutoService(Processor.class)
public class AnnotationCompiler extends AbstractProcessor {

    //生成文件的对象
    Filer filer;

    /**
     * 声明支持的java版本
     *
     * @return
     */
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return processingEnv.getSourceVersion();
    }

    /**
     * 声明要处理的注解是哪些
     *
     * @return
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> types = new HashSet<>();
        types.add(BindView.class.getName());
        return types;
    }

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        filer = processingEnv.getFiler();
    }

    /**
     * 核心方法 这个方法能够得到注解所标记的内容
     *
     * @param set
     * @param roundEnvironment
     * @return
     */
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        // 获取到当前模块中所有用到了BindView的节点
        Set<? extends Element> elementsAnnotatedWithBindView = roundEnvironment.getElementsAnnotatedWith(BindView.class);


        //TypeElement 类节点
        //ExecutableElement 方法节点
        //VariableElement  成员变量节点

        //每个activity类中被标记的节点区分开来然后统一存储起来 键为activityName 值为activity中所有被标记的变量
        Map<String, List<VariableElement>> map = new HashMap<>();
        for (Element element : elementsAnnotatedWithBindView) {
            //转换一下 得到的是成员变量的节点
            VariableElement variableElement = (VariableElement) element;
            // 获取类名
            TypeElement typeElement = (TypeElement) variableElement.getEnclosingElement();
            String activityName = typeElement.getSimpleName().toString();
            //@BindView注解过的元素做一个规整，以enclosingElement为键的原因就是把在同一个类中的被@BindView注解过的元素放到一起，
            List<VariableElement> variableElements = map.get(activityName);
            if (variableElements == null) {
                variableElements = new ArrayList<>();
                map.put(activityName, variableElements);
            }
            variableElements.add(variableElement);
        }

        //开始生成xxxActivity$$ViewBinder.java文件
        if (map.size() > 0) {

            Writer writer = null;
            //得到map的key的迭代器
            Iterator<String> iterator = map.keySet().iterator();
            while (iterator.hasNext()) {
                String activityName = iterator.next();
                String newBinderName = activityName + "$$ViewBinder";
                // 得到value
                List<VariableElement> variableElements = map.get(activityName);
                String packageName = getPackageName(variableElements.get(0));
                try {
                    JavaFileObject sourceFile = filer.createSourceFile(packageName + "." + activityName + "$$ViewBinder");
                    writer = sourceFile.openWriter();
                    StringBuffer stringBuffer = new StringBuffer();
                    // 导入包名
                    stringBuffer.append("package " + packageName + ";\n");
                    // 导入依赖
                    stringBuffer.append("import android.view.View;\n");
                    // 类名
                    stringBuffer.append("public class " + newBinderName + "{\n");
                    // 构造函数
                    stringBuffer.append("public " + newBinderName + "(final " + packageName + "." + activityName + " target){\n");
                    //遍历value值
                    for (VariableElement element : variableElements) {
                        //获取到成员变量的名字
                        String fieldName = element.getSimpleName().toString();
                        //得到类型 强转的时候用
                        TypeMirror typeMirror = element.asType();
                        //获取到注解中的id
                        int resId = element.getAnnotation(BindView.class).value();
                        // 生成 findViewById
                        stringBuffer.append("target." + fieldName + " =(" + typeMirror + ") target.findViewById(" + resId + ");\n");
                    }
                    stringBuffer.append("}\n}");
                    writer.write(stringBuffer.toString());
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    if (writer != null) {
                        try {
                            writer.close();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }

        return false;
    }

    /**
     * 获取包名的方法
     *
     * @param variableElement
     * @return
     */
    public String getPackageName(VariableElement variableElement) {
        TypeElement typeElement = (TypeElement) variableElement.getEnclosingElement();
        PackageElement packageElement = processingEnv.getElementUtils().getPackageOf(typeElement);
        return packageElement.getQualifiedName().toString();
    }
}
```

### 4.使用以及生成的TwoActivity$$ViewBinder文件（全局搜索便可以得到）

```java
package com.yunxiao.dev.mybutterknife;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.widget.Button;

import com.yunxiao.dev.annotation.BindView;

public class TwoActivity extends AppCompatActivity {

    @BindView(R.id.button_1)
    Button button1;
    @BindView(R.id.button_2)
    Button button2;
    @BindView(R.id.button_3)
    Button button3;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_two);
        button1.setText("We");
        button2.setText("Made");
        button3.setText("It");
    }
}
```

```java
package com.yunxiao.dev.mybutterknife;
import android.view.View;
public class TwoActivity$$ViewBinder{
public TwoActivity$$ViewBinder(final com.yunxiao.dev.mybutterknife.TwoActivity target){
target.button1 =(android.widget.Button) target.findViewById(2131230824);
target.button2 =(android.widget.Button) target.findViewById(2131230825);
target.button3 =(android.widget.Button) target.findViewById(2131230826);
}
}
```

### 5.效果图片

![](BK效果图.png)

可以看到我们绑定控件的效果实现了。

我们的分析ButterKnife系列文章也就结束了。本文大量参考了[顾修忠](https://blog.csdn.net/ta893115871)的分析butterKnife的系列文章。感谢各位前人的努力。如果各位有兴趣和我一起探讨Android技术的话欢迎加我的微信 GCRAlice716。各位再见~。