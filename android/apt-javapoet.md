---
title: APT和JavaPoet
tags: apt javapoet
categories: Android
---

* TOC
{:toc}

# 1、基本概念

### 1.1、APT(Annotation Processing Tool)
是一种处理注释的工具，它对源代码文件进行检测找出其中的Annotation，根据注解自动生成代码。如果想要自定义的注解处理器能够正常运行，必须要通过APT工具来进行处理。也可以这样理解，只有通过声明APT工具后，程序在编译期间自定义注解解释器才能执行。比如ButterKnife、EventBus.

通俗理解：根据规则，帮我们生成代码、生成类文件。

### 1.2、java是一种结构体语言
```java
package com.pioneer.networkdemo;  //PackageElement 包元素/节点

public class Main {               //TypeElement 类元素/节点

    private int x;                //VariableElement 属性元素/节点

    private void main(){          //ExecuteableElement 方法元素/节点
    }
}
```

### 1.3、API
![api](../static/img/javapoet.png)

### 1.4、注解的写法
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface ARouter {
    String path();
}
```

> @Target(ElementType.TYPE)  // 接口、类、枚举、注解
> @Target(ElementType.FIELD) // 属性、枚举的常量
> @Target(ElementType.METHOD) // 方法
> @Target(ElementType.PARAMETER) // 方法参数
> @Target(ElementType.CONSTRUCTOR) // 构造方法
> @Target(ElementType.LOCAL_VARIBLE) // 局部变量
> @Target(ElementType.ANNOTATION_TYPE) // 该注解使用在另一个注解上
> @Target(ElementType.PACKAGE) // 包

> @Retention(RetentionPolicy.CLASS) // 编译时进行一些预处理操作，注解会在class文件中存在
> @Retention(RetentionPolicy.RUNTIME) // 注解会在class字节码中存在，jvm加载时可以通过反射获取到该注解的内容

RetentionPolicy使用规则：
> 
>- 一般如果需要在运动时去动态获取注解信息，用RUNTIME注解；
>- 要在编译时进行一些预处理操作，如ButterKnife，用class注解，注解会在class文件中存在，但是在运行时会被丢弃；
>- 做一些检查性的操作，如@Override，用SOURCE源码注解，注解仅存在源码级别，在编译的时候丢弃该注解。

# 2、APT的常规写法

### 2.1、创建annotation库

### 2.2、创建compiler库，引入annotation库和APT服务需要的库

```java
apply plugin: 'java-library'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    // 谷歌注解处理器服务
    compileOnly 'com.google.auto.service:auto-service:1.0-rc4'
    annotationProcessor 'com.google.auto.service:auto-service:1.0-rc4'

    // 引入annotation注解项目库，处理注解
    implementation project(':annotation')
}

// 解决java控制台输出乱码问题
tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
}

sourceCompatibility = "7"
targetCompatibility = "7"
```

### 2.3、在使用的module中引用

```java
    // 引入annotation注解项目库，处理注解
    implementation project(':annotation')
    // 引入注解处理器
    annotationProcessor project(':compiler')
```

# 3、ARouter案例

#### 3.1、在annotation中创建注解类

```java
@Target(ElementType.TYPE) // 作用在类上
@Retention(RetentionPolicy.CLASS) // 编译时进行一些预处理操作，
public @interface ARouter {
    String path();
}
```

#### 3.2、注解处理器

```java
public class ARouterProcessor extends AbstractProcessor {

    private Elements elementUtils;
    private Messager messager;
    private Filer filer;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        elementUtils = processingEnv.getElementUtils();
        messager = processingEnv.getMessager();
        filer = processingEnv.getFiler();
    }

    /**
     * 1、接受参数
     */
    @Override
    public Set<String> getSupportedOptions() {
        return super.getSupportedOptions();
    }

    /**
     * 2、支持注解类型，process会处理支持的注解
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return super.getSupportedAnnotationTypes();
    }

    /**
     * 3、指定JDK的编译版本
     */
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return super.getSupportedSourceVersion();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        return false;
    }
}
```

这种写法太啰嗦，给注解处理器ARouterProcessor添加注解的方式实现上面3个功能：

```java
// 用来生成 META-INF /services/javax.annotation.processing.Processor文件
@AutoService(Processor.class)
// 支持的注解类型
@SupportedAnnotationTypes({Constants.AROUTER_ANNOTATION_TYPE})
// 指定JDK编译版本
@SupportedSourceVersion(SourceVersion.RELEASE_7)
// 接受参数
@SupportedOptions(value = {"xxx", "xxx"})
```

#### 3.3、根据注解生成代码

```java
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        if (annotations.isEmpty()) {
            messager.printMessage(Diagnostic.Kind.NOTE, "annotations.isEmpty()");
            return false;
        }
        // 获取所有带ARouter注解的类节点
        Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(ARouter.class);
        // 遍历所有类节点
        for (Element element : elements) {
            // 通过类节点获取包节点
            String packageName = elementUtils.getPackageOf(element).getQualifiedName().toString();
            // 获取简单类名
            String className = element.getSimpleName().toString();
            messager.printMessage(Diagnostic.Kind.NOTE, "被注解的类有：" + packageName + "." + className);
            // 最终生成的类文件名
            String finalClassName = className + "$ARouter";

            try {
                // 创建一个新的源文件
                JavaFileObject sourceFile = filer.createClassFile(packageName + "." + finalClassName);
                BufferedWriter writer = new BufferedWriter(sourceFile.openWriter());
                // 设置报名
                writer.write("package " + packageName + ";\n");
                writer.write("public class " + finalClassName + " {\n");
                writer.write("public void hello(String path){\n}\n}");
                writer.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return true;
    }
```

# 4、[JavaPoet](https://github.com/square/javapoet)

上面是通过拼接字符串的方式生成代码，JavaPoet是通过Java代码来实现代码生成。

#### 4.1、JavaPoet的8个常用类

![api](../static/img/javapoet_8_class.png)


#### 4.2、JavaPoet字符串格式化规则

![api](../static/img/javapoet_format_string.png)


```java
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        // 创建方法
        MethodSpec main = MethodSpec.methodBuilder("main")
                .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
                .addParameter(String[].class, "args")
                .addStatement("$T.out.println($S)", System.class, "Hello JavaPoet")
                .build();
        //创建类
        TypeSpec typeSpec = TypeSpec.classBuilder("HelloWord")
                .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
                .addMethod(main)
                .build();

        //创建Java文件
        JavaFile file = JavaFile.builder("com.pioneer.apt.demo", typeSpec)
                .build();

        try {
            file.writeTo(filer);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }
```








