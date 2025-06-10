# Scala Reflection

> 反射分为编译时反射和运行时反射。编译时反射拥有修改AST Tree的能力，一般用不到。这里只关注运行时反射。
>
> [概览 | Reflection | Scala Documentation (scala-lang.org)](https://docs.scala-lang.org/zh-cn/overviews/reflection/overview.html)

运行时反射指的是在运行时对类型或者对象实例的操作，有如下三种：

- 解析包括泛型在内的各种对象的类型
- 实例化一个对象
- 能去访问或者调用对象中的成员

## 三种操作

### 解析类型

官网给的例子核心是：`def getTypeTag[T:ru.TypeTag](obj:T) = ru.typeTag[T]`，它将`T`信息从以`TypeTag`形式从编译时带到运行时，只需要将对象传入，即可在运行时获取其真实类型。并可以通过`getTypeTag(obj).tpe.decls`获取全量类型信息，比如构造函数，伴生对象，方法定义。

### 实例构建

> 这里使用的是调用器镜像，其最终暴露给用户构造器用于构造实例。

第一步，使用`val m = ru.runtimeMirror(getClass.getClassLoader)`，将所有的类全部加载(不止是某个类)。

第二步，使用`val classPerson = ru.typeOf[Person].typeSymbol.asClass`，将Person的类型信息获取到，并拿到类型的定义Symbol，再转换为ClassSymbol，而不是其他类型符号。

> 需要注意ru.typeOf[C.type].termSymbol.asModule 和Mirror.staticModule("pkg.C") 的不同，前者需要编译时有C的伴生对象，而后者不需要，只需要全类名存在就可以。所以如果实现插件系统，一般需要使用后面的例子。

第三步，使用`val cm = m.reflectClass(classPerson)`，使用运行时镜像，传入某个类的ClassSymbol，则可以得到这个类的ClassMirror。此镜像可以用于获取构造方法，创建实例，访问字段等。

第四步，使用`val ctor ru.typeOf[Person].decl(ru.termNames.CONSTRUCTOR).asMethod`，将构造器Symbol拿到，然后通过`val ctorm = cm.reflectConstructor(ctor)`拿到调用器镜像。这里的调用器镜像可以直接传入参数，得到对应实例，比如`val p = ctorm("Mike")`。

### 访问运行时的成员

> 这里使用的同样是调用器镜像。

和实例构建一样，同样使用的是调用器镜像。

流程同样类似：`m -> Term Symbol -> Term m- > Field`

### 核心概念

#### universe

反射的根据时机不同，分为编译时反射和运行时反射。其通过`scala.reflect.runtime.universe`包中的成员来区别，一般使用的是runtime部分的。

#### Mirror

Mirror，分为类加载器镜像，和调用者镜像。从前者可以获取后者。

- 类加载器镜像：用于将名称转换为Symbol
- 调用者镜像，通过Symbol得到调用者镜像，用于实现真正的反射调用需求。
