---
layout: post
uuid: 9d3948f9-d931-4b0f-be1d-07d6b07b7e03
title: 技术：重复造轮子-Lombok（Processer 与 IDEA PSI入门）
categories: [技术]
description: 重复造轮子之Lombok篇
keywords: 技术, lombok, processer, 重复造轮子
---

之前做Android开发的时候，就用了很久的lombok，但是一直没有仔细研究它的原理。最近有时间自学一些东西，我选择了自己实现一个简单的注解生成器，用于生成get与set方法。

## 源码

源码放在了github上，一下内容配合源码食用口味更好。

<https://github.com/Tconan99/DenoProcesser>

<https://github.com/Tconan99/DenoPlugin>

## Java Processer

根据查询到的资料，Processer可以在JDK编译代码的时候被调用执行，修改Java生成的语法书，最后生成的class就会被改变。（class文件就是根据语法树生成的）

![](https://zihuatanejo.top/images/blog02/注解代码编译效果.png)

这个图片就是效果图，下面就是核心源码，其实整个过程的逻辑并不复杂。这个process方法会在jdk编译时被调用，然后判断这类是不是被注解标记了，如果标记了就遍历语法树找到属性变量，然后生成get和set方法插入语法树里面。没错就这么简单，但是花了我好久的时间才抄出来。

```js
@Override
public synchronized boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    Set<? extends Element> set = roundEnv.getElementsAnnotatedWith(ConanData.class);
    set.forEach(element -> {
        JCTree jcTree = trees.getTree(element);
        jcTree.accept(new TreeTranslator() {
            @Override
            public void visitClassDef(JCTree.JCClassDecl jcClassDecl) {
                List<JCTree.JCVariableDecl> jcVariableDeclList = List.nil();

                for (JCTree tree : jcClassDecl.defs) {
                    if (tree.getKind().equals(Tree.Kind.VARIABLE)) {
                        JCTree.JCVariableDecl jcVariableDecl = (JCTree.JCVariableDecl) tree;
                        jcVariableDeclList = jcVariableDeclList.append(jcVariableDecl);
                    }
                }

                jcVariableDeclList.forEach(jcVariableDecl -> {
                    messager.printMessage(Diagnostic.Kind.NOTE, jcVariableDecl.getName() + " has been processed");
                    jcClassDecl.defs = jcClassDecl.defs.prepend(makeGetterMethodDecl(jcVariableDecl));
                    jcClassDecl.defs = jcClassDecl.defs.prepend(makeSetterMethodDecl(jcVariableDecl));
                });
                super.visitClassDef(jcClassDecl);
            }

        });
    });

    return true;
}

private JCTree.JCMethodDecl makeGetterMethodDecl(JCTree.JCVariableDecl jcVariableDecl) {
    ListBuffer<JCTree.JCStatement> statements = new ListBuffer<>();
    statements.append(treeMaker.Return(treeMaker.Select(treeMaker.Ident(names.fromString("this")), jcVariableDecl.getName())));
    JCTree.JCBlock body = treeMaker.Block(0, statements.toList());
    return treeMaker.MethodDef(treeMaker.Modifiers(Flags.PUBLIC), getMethodName(jcVariableDecl.getName(), "get"), jcVariableDecl.vartype, List.nil(), List.nil(), List.nil(), body, null);
}

private JCTree.JCMethodDecl makeSetterMethodDecl(JCTree.JCVariableDecl jcVariableDecl) {
    JCTree.JCVariableDecl param = treeMaker.VarDef(treeMaker.Modifiers(Flags.PARAMETER), jcVariableDecl.name, jcVariableDecl.vartype, null);
    param.pos = jcVariableDecl.pos;
    List<JCTree.JCVariableDecl> parameters = List.of(param);

    ListBuffer<JCTree.JCStatement> statements = new ListBuffer<>();
    statements.append(treeMaker.Exec(treeMaker.Assign(treeMaker.Select(treeMaker.Ident(names.fromString("this")), jcVariableDecl.getName()), treeMaker.Ident(jcVariableDecl.getName()))));
    JCTree.JCBlock body = treeMaker.Block(0, statements.toList());

    return treeMaker.MethodDef(treeMaker.Modifiers(Flags.PUBLIC), getMethodName(jcVariableDecl.getName(), "set"), treeMaker.Type(new Type.JCVoidType()), List.nil(), parameters, List.nil(), body, null);
}
```

需要注意这行代码，不写会报错，卡了我两天时间。

```js
param.pos = jcVariableDecl.pos;
```

## IDEA 内的报错

以上代码写完后，编译生成的class就有get和set方法了。然而，在IDEA内会出现提示这个方法不存的问题，也没有代码提醒。

![](https://zihuatanejo.top/images/blog02/Processer错误提示.png)

但是他能运行成功哦！

![](https://zihuatanejo.top/images/blog02/Processer运行成功.png)

为什么报错还能运行呢？这个地方我疑惑了很久，最后想明白，IDE报错和运行代码是两个逻辑，只要class没有问题，那就能运行。而报错是IDE检查代码的逻辑，lombok的插件就是解决了报错问题。

而报错的原因我思考了很久，才想明白，IDEA不用读取class文件的内容来判断方法是否存在的。他是直接读取java文件的，但是我也不能直接修改java文件呀。

## lombok plugin 

Bingo！IDEA是把文件内容转化成 PSI 语法树的，说到这大家可能就明白了，我们需要写个插件影响 PSI 生成的过程，修改语法树达到不报错的目的。其实这个地方跟 Processer 原理是一样，就是一层窗户纸，想明白了就感觉很简单。

这个地方，网上也有人问问题，但是没有答案。

最开始，我是好奇 lombok plugin 做了什么，结果我换了巨多的关键字，都没有找到答案。网上的资料啊只介绍了 Processer 的过程，几乎没有介绍这个地方的。

然后，我翻了 IDEA 官网的文档，找到了英文文档，里面有讲解PSI的篇章，但是太晦涩难懂了，而且我还不知道啥是PSI。

最后，机智的我去翻了lombok plugin源码。最开始我是翻过的，看不懂。。。现在也看不懂。这次不一样，我直接翻看了第一个版本的源码，没错！就是第一次提交的代码，这次就看懂了。

以下是代码，都是抄的。。。

```js
@NotNull
@Override
protected <Psi extends PsiElement> List<Psi> getAugments(@NotNull PsiElement element, @NotNull Class<Psi> type) {

    if (!(element instanceof PsiClass) || !element.isValid() || !element.isPhysical()) {
        return Collections.emptyList();
    }

    final List<Psi> result = new ArrayList<>();
    final PsiClass psiClass = (PsiClass) element;
    // System.out.println("Called for class: " + psiClass.getQualifiedName() + " type: " + type.getName());

    if (type.isAssignableFrom(PsiMethod.class)) {
        // System.out.println("collect methods of class: " + psiClass.getQualifiedName());
        processPsiClassAnnotations(result, psiClass, type);
        // processPsiClassFieldAnnotation(result, psiClass, type);
    }

    return result;
}

private <Psi extends PsiElement> void processPsiClassAnnotations(@NotNull List<Psi> result, @NotNull PsiClass psiClass, @NotNull Class<Psi> type) {
    // System.out.println("Processing class annotations BEGINN: " + psiClass.getQualifiedName());

    final PsiModifierList modifierList = psiClass.getModifierList();
    if (modifierList != null) {
        for (PsiAnnotation psiAnnotation : modifierList.getAnnotations()) {
            processClassAnnotation(psiAnnotation, psiClass, result, type);
        }
    }
    // System.out.println("Processing class annotations END: " + psiClass.getQualifiedName());
}

private <Psi extends PsiElement> void processClassAnnotation(@NotNull PsiAnnotation psiAnnotation, @NotNull PsiClass psiClass, @NotNull List<Psi> result, @NotNull Class<Psi> type) {
    final String annotationName = StringUtil.notNullize(psiAnnotation.getQualifiedName()).trim();
    final String supportedAnnotation = ConanData.class.getName();
    if ((supportedAnnotation.equals(annotationName) || supportedAnnotation.endsWith(annotationName)) &&
            (type.isAssignableFrom(PsiMethod.class))) {
        process(psiClass, psiAnnotation, result);
    }
}

private <Psi extends PsiElement> void process(@NotNull PsiClass psiClass, @NotNull PsiAnnotation psiAnnotation, @NotNull List<Psi> target) {
    for (PsiField psiField : psiClass.getFields()) {
//            boolean createSetter = true;
//            PsiModifierList modifierList = psiField.getModifierList();
//            if (null != modifierList) {
//                createSetter = !modifierList.hasModifierProperty(PsiModifier.STATIC); // 处理static
//                createSetter &= !hasFieldProcessorAnnotation(modifierList); // 处理 属性也添加注解的情况防止重复执行，我猜的 ^-^
//            }

        process(psiField, psiAnnotation, target);
    }
}

public <Psi extends PsiElement> void process(@NotNull PsiField psiField, @NotNull PsiAnnotation psiAnnotation, @NotNull List<Psi> target) {
    boolean result = false;

    final String visibility = LombokProcessorUtil.getMethodVisibity(psiAnnotation);
    if (null != visibility) {
        try {
            Project project = psiField.getProject();

            PsiClass psiClass = psiField.getContainingClass();
            PsiManager manager = psiField.getContainingFile().getManager();
            PsiElementFactory elementFactory = JavaPsiFacade.getInstance(project).getElementFactory();

            String fieldName = psiField.getName();
            PsiType psiType = psiField.getType();
            String typeName = psiType.getCanonicalText();
            // String getterName = TransformationsUtil.toGetterName(fieldName, psiType.equalsToText("boolean"));
            String commontName = fieldName.substring(0, 1).toUpperCase() + fieldName.substring(1, fieldName.length());

            String getterName = "get" + commontName;
            final PsiMethod getterValuesMethod = elementFactory.createMethodFromText(
                    visibility + typeName + " " + getterName + "() { return this." + fieldName + ";}",
                    psiClass);
            target.add((Psi) new MyLightMethod(manager, getterValuesMethod, psiClass));
            // result = true;

            String setterName = "set" + commontName;
            final PsiMethod setterValuesMethod = elementFactory.createMethodFromText(
                    visibility + "void " + setterName + "(" + typeName + " " + fieldName + ") { return this." + fieldName + " = " + fieldName + ";}",
                    psiClass);
            target.add((Psi) new MyLightMethod(manager, setterValuesMethod, psiClass));


        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    // psiField.putUserData(READ_KEY, result);
}
```
逻辑依旧很简单，判断类是不是有注解，然后生成对应方法即可。

这个写法没有注重效率，可以配合缓存使用，如果之前生成过，那就用缓存就可以，如果后续有精力我会继续改进。

以上就是这全部内容了，是不是很简单？我写了半个多月。。。规划了半年。。。

by 费城的二鹏 2019.07.08 写于公司
