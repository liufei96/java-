# 手写spring的IOC容器

<!-- toc -->

对应的是day0019的代码

## XML技术

它是可扩展标记语言（Extensible Markup Language，简称XML），是一种标记语言。

XML 全称为可扩展的标记语言。主要用于描述数据和用作配置文件。

XML 文档在逻辑上主要由一下 5 个部分组成：

XML 声明：指明所用 XML 的版本、文档的编码、文档的独立性信息

文档类型声明：指出 XML 文档所用的 DTD

元素：由开始标签、元素内容和结束标签构成

注释：以结束，用于对文档中的内容起一个说明作用

处理指令：通过处理指令来通知其他应用程序来处理非 XML 格式的数据，格式为

XML 文档的根元素被称为文档元素，它和在其外部出现的处理指令、注释等作为文档实体的子节点，根元素本身和其内部的子元素也是一棵树。

```xml
<students>  
    <student1 id="001">  
        <微信公众号>@残缺的孤独</微信公众号>  
        <学号>20140101</学号>  
        <地址>北京海淀区</地址>  
        <座右铭>要么强大，要么听话</座右铭>  
    </student1>  
    <student2 id="002">  
        <新浪微博>@残缺的孤独</新浪微博>  
        <学号>20140102</学号>  
        <地址>北京朝阳区</地址>  
        <座右铭>在哭泣中学会坚强</座右铭>  
    </student2>  
</students>  
```

<?xml version="1.0" encoding="UTF-8"?>\***

作用[xml文件](https://www.baidu.com/s?wd=xml文件&tn=44039180_cpr&fenlei=mv6quAkxTZn0IZRqIHckPjm4nH00T1YdnjK9rjbvuWfLPAN9Ph7W0ZwV5Hcvrjm3rH6sPfKWUMw85HfYnjn4nH6sgvPsT6KdThsqpZwYTjCEQLGCpyw9Uz4Bmy-bIi4WUvYETgN-TLwGUv3EPHR4rjR1rHc4nWTYP10krj0Y)头部要写的话，说明了xml的版本和编码，utf-8一般是[网络传输](https://www.baidu.com/s?wd=网络传输&tn=44039180_cpr&fenlei=mv6quAkxTZn0IZRqIHckPjm4nH00T1YdnjK9rjbvuWfLPAN9Ph7W0ZwV5Hcvrjm3rH6sPfKWUMw85HfYnjn4nH6sgvPsT6KdThsqpZwYTjCEQLGCpyw9Uz4Bmy-bIi4WUvYETgN-TLwGUv3EPHR4rjR1rHc4nWTYP10krj0Y)用的编码

### XML解析方式？

Dom4j、Sax、Pull

### Dom4j和Sax区别？

dom4j不适合大文件的解析，因为它是一下子将文件加载到内存中，所以有可能出现内存溢出，sax是基于事件来对xml进行解析的，所以他可以解析大文件的xml，也正是因为如此，所以dom4j可以对xml进行灵活的增删改查和导航，而sax没有这么强的灵活性，所以sax经常是用来解析大型xml文件，而要对xml文件进行一些灵活（crud）操作就用dom4j。

### 使用dom4j解析xml

```java
public class XmlUtils {

    public static void main(String[] args) throws DocumentException {
        XmlUtils xmlUtils = new XmlUtils();
        xmlUtils.test001();
    }

    public void test001() throws DocumentException {
        SAXReader saxReader = new SAXReader();
        Document read = saxReader.read(getClassPath("student.xml"));
        // 获取根节点
        Element rootElement = read.getRootElement();
        getNodes(rootElement);
    }

    public static InputStream getClassPath(String xmlPath) {
        InputStream resourceAsStream = XmlUtils.class.getClassLoader().getResourceAsStream(xmlPath);
        return resourceAsStream;
    }

    public static void getNodes(Element rootElement) {
        System.out.println("获取当前名称:" + rootElement.getName());
        // 获取属性信息
        List<Attribute> attributes = rootElement.attributes();
        for (Attribute attribute : attributes) {
            System.out.println("属性:" + attribute.getName() + "---" + attribute.getText());
        }
        // 获取属性value
        String value = rootElement.getTextTrim();
        if (!StringUtils.isEmpty(value)) {
            System.out.println("value:" + value);
        }
        // 使用迭代器遍历,继续遍历子节点
        Iterator<Element> elementIterator = rootElement.elementIterator();
        while (elementIterator.hasNext()) {
            Element next = elementIterator.next();
            getNodes(next);
        }
    }

    // 读取xml配置文件
    public static List<Element> readerXml(String xmlPath) throws DocumentException {
        SAXReader saxReader = new SAXReader();
        if (StringUtils.isBlank(xmlPath)) {
            new Exception("xml路径不能为空");
        }
        Document root = saxReader.read(getClassPath(xmlPath));
        // 获取根节点
        Element rootElement = root.getRootElement();
        List<Element> elements = rootElement.elements();
        if (elements == null || elements.isEmpty()) {
            return null;
        }
        return elements;
    }

    public static String findXmlByIDClass(List<Element> elements, String beanId) throws Exception {
        for (Element element : elements) {
            String beanIdValue = element.attributeValue("id");
            if (beanIdValue == null) {
                throw new Exception("使用该beanId没有查找到对应的元素");
            }
            if (!beanIdValue.equals(beanId)) {
                continue;
            }
            // 获取Class地址属性
            String classPath = element.attributeValue("class");
            if (StringUtils.isNotBlank(classPath)) {
                return classPath;
            }
        }
        return null;
    }
}
```

### XML与JSON区别

Xml是重量级数据交换格式，占宽带比较大。

JSON是轻量级交换格式，xml占宽带小。

所有很多互联网公司都会使用json作为数据交换格式

很多银行项目，大多数还是在使用xml。

## SpringIOC的原理

使用反射机制+XML技术

### 手写SpringIOC XML版本

> 思想：
>
> 1. 读取xml配置文件
>
> 2. 使用beanId查找对应的class对象
>
> 3. 获取class的信息地址，使用反射进行初始化

XmlUtils工具类

```java
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:aop="http://www.springframework.org/schema/aop" xmlns:tx="http://www.springframework.org/schema/tx"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
    	 http://www.springframework.org/schema/beans/spring-beans.xsd
     	 http://www.springframework.org/schema/context
         http://www.springframework.org/schema/context/spring-context.xsd
         http://www.springframework.org/schema/aop
         http://www.springframework.org/schema/aop/spring-aop.xsd
         http://www.springframework.org/schema/tx
     	 http://www.springframework.org/schema/tx/spring-tx.xsd">

	<bean id="user" class="com.liufei.entity.User"></bean>

</beans>
```

```java
public class MyClassPathXmlApplicationContext {

    private String xmlPath;

    public MyClassPathXmlApplicationContext(String xmlPath) {
        this.xmlPath = xmlPath;
    }

    public Object getBean(String beanId) throws Exception {
        // 1.读取配置文件
        List<Element> elements = XmlUtils.readerXml(xmlPath);
        if (elements == null) {
            throw new Exception("该配置文件没有子元素");
        }
        // 2.使用beanId查找对应的class对象
        String xmlByIDClass = XmlUtils.findXmlByIDClass(elements, beanId);
        if (xmlByIDClass == null) {
            throw new Exception("没有找到改beanId对应的class");
        }
        // 3. 获取class的信息地址，使用反射进行初始化
        Class<?> aClass = Class.forName(xmlByIDClass);
        return aClass.newInstance();
    }
}
```

```java
public class Test001 {
    public static void main(String[] args) throws Exception {
        MyClassPathXmlApplicationContext applicationContext = new MyClassPathXmlApplicationContext("spring.xml");
        User user = (User) applicationContext.getBean("user");
        String userName = user.getUserName();
        System.out.println(userName);
    }
}
```

### 手写SpringIOC注解版本版本

> 思想：（扫包）
>
> 1. 使用java反射机制扫包，获取当前包下的所有类
> 2. 判断类上是否有我们定义的注解
> 3. 然后将其存放到ConcurrentHashMap<>中，beanId为key，Class内容为value

```java
public class ClassUtil {

	/**
	 * 取得某个接口下所有实现这个接口的类
	 */
	public static List<Class> getAllClassByInterface(Class c) {
		List<Class> returnClassList = null;

		if (c.isInterface()) {
			// 获取当前的包名
			String packageName = c.getPackage().getName();
			// 获取当前包下以及子包下所以的类
			List<Class<?>> allClass = getClasses(packageName);
			if (allClass != null) {
				returnClassList = new ArrayList<Class>();
				for (Class classes : allClass) {
					// 判断是否是同一个接口
					if (c.isAssignableFrom(classes)) {
						// 本身不加入进去
						if (!c.equals(classes)) {
							returnClassList.add(classes);
						}
					}
				}
			}
		}

		return returnClassList;
	}

	/*
	 * 取得某一类所在包的所有类名 不含迭代
	 */
	public static String[] getPackageAllClassName(String classLocation, String packageName) {
		// 将packageName分解
		String[] packagePathSplit = packageName.split("[.]");
		String realClassLocation = classLocation;
		int packageLength = packagePathSplit.length;
		for (int i = 0; i < packageLength; i++) {
			realClassLocation = realClassLocation + File.separator + packagePathSplit[i];
		}
		File packeageDir = new File(realClassLocation);
		if (packeageDir.isDirectory()) {
			String[] allClassName = packeageDir.list();
			return allClassName;
		}
		return null;
	}

	/**
	 * 从包package中获取所有的Class
	 * @param packageName
	 * @return
	 */
	public static List<Class<?>> getClasses(String packageName) {

		// 第一个class类的集合
		List<Class<?>> classes = new ArrayList<Class<?>>();
		// 是否循环迭代
		boolean recursive = true;
		// 获取包的名字 并进行替换
		String packageDirName = packageName.replace('.', '/');
		// 定义一个枚举的集合 并进行循环来处理这个目录下的things
		Enumeration<URL> dirs;
		try {
			dirs = Thread.currentThread().getContextClassLoader().getResources(packageDirName);
			// 循环迭代下去
			while (dirs.hasMoreElements()) {
				// 获取下一个元素
				URL url = dirs.nextElement();
				// 得到协议的名称
				String protocol = url.getProtocol();
				// 如果是以文件的形式保存在服务器上
				if ("file".equals(protocol)) {
					// 获取包的物理路径
					String filePath = URLDecoder.decode(url.getFile(), "UTF-8");
					// 以文件的方式扫描整个包下的文件 并添加到集合中
					findAndAddClassesInPackageByFile(packageName, filePath, recursive, classes);
				} else if ("jar".equals(protocol)) {
					// 如果是jar包文件
					// 定义一个JarFile
					JarFile jar;
					try {
						// 获取jar
						jar = ((JarURLConnection) url.openConnection()).getJarFile();
						// 从此jar包 得到一个枚举类
						Enumeration<JarEntry> entries = jar.entries();
						// 同样的进行循环迭代
						while (entries.hasMoreElements()) {
							// 获取jar里的一个实体 可以是目录 和一些jar包里的其他文件 如META-INF等文件
							JarEntry entry = entries.nextElement();
							String name = entry.getName();
							// 如果是以/开头的
							if (name.charAt(0) == '/') {
								// 获取后面的字符串
								name = name.substring(1);
							}
							// 如果前半部分和定义的包名相同
							if (name.startsWith(packageDirName)) {
								int idx = name.lastIndexOf('/');
								// 如果以"/"结尾 是一个包
								if (idx != -1) {
									// 获取包名 把"/"替换成"."
									packageName = name.substring(0, idx).replace('/', '.');
								}
								// 如果可以迭代下去 并且是一个包
								if ((idx != -1) || recursive) {
									// 如果是一个.class文件 而且不是目录
									if (name.endsWith(".class") && !entry.isDirectory()) {
										// 去掉后面的".class" 获取真正的类名
										String className = name.substring(packageName.length() + 1, name.length() - 6);
										try {
											// 添加到classes
											classes.add(Class.forName(packageName + '.' + className));
										} catch (ClassNotFoundException e) {
											e.printStackTrace();
										}
									}
								}
							}
						}
					} catch (IOException e) {
						e.printStackTrace();
					}
				}
			}
		} catch (IOException e) {
			e.printStackTrace();
		}

		return classes;
	}

	/**
	 * 以文件的形式来获取包下的所有Class
	 * 
	 * @param packageName
	 * @param packagePath
	 * @param recursive
	 * @param classes
	 */
	public static void findAndAddClassesInPackageByFile(String packageName, String packagePath, final boolean recursive,
			List<Class<?>> classes) {
		// 获取此包的目录 建立一个File
		File dir = new File(packagePath);
		// 如果不存在或者 也不是目录就直接返回
		if (!dir.exists() || !dir.isDirectory()) {
			return;
		}
		// 如果存在 就获取包下的所有文件 包括目录
		File[] dirfiles = dir.listFiles(new FileFilter() {
			// 自定义过滤规则 如果可以循环(包含子目录) 或则是以.class结尾的文件(编译好的java类文件)
			public boolean accept(File file) {
				return (recursive && file.isDirectory()) || (file.getName().endsWith(".class"));
			}
		});
		// 循环所有文件
		for (File file : dirfiles) {
			// 如果是目录 则继续扫描
			if (file.isDirectory()) {
				findAndAddClassesInPackageByFile(packageName + "." + file.getName(), file.getAbsolutePath(), recursive,
						classes);
			} else {
				// 如果是java类文件 去掉后面的.class 只留下类名
				String className = file.getName().substring(0, file.getName().length() - 6);
				try {
					// 添加到集合中去
					classes.add(Class.forName(packageName + '.' + className));
				} catch (ClassNotFoundException e) {
					e.printStackTrace();
				}
			}
		}
	}
}
```

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface ExService {
}
```

```java
public class ExClassPathXmlApplicationContext {

    // 扫包范围
    private String packageName;

    private ConcurrentHashMap<String, Class<?>> beans = null;

    public ExClassPathXmlApplicationContext(String packageName) throws Exception {
        beans = new ConcurrentHashMap<>();
        this.packageName = packageName;
        initBeans();
    }

    // 初始化对象
    public  void initBeans() throws Exception {
        // 1.使用java反射机制扫包，获取当前包下的所有类
        List<Class<?>> classes = ClassUtil.getClasses(packageName);
        // 2. 判断类上是否存在注入bean的注解
        ConcurrentHashMap<String, Class<?>> classExistAnnotation = findClassExistAnnotation(classes);
        if (classExistAnnotation == null || classExistAnnotation.isEmpty()) {
            throw new Exception("改包下没有任何类的注解");
        }
    }

    public Object getBean(String beanId) throws Exception {
        if (StringUtils.isBlank(beanId)) {
            throw new Exception("beanId不能为空");
        }
        Class<?> aClass = beans.get(beanId);
        if (aClass == null) {
            throw new Exception("class not found");
        }
        return aClass.newInstance();
    }

    // 判断类上是否有注入bean的注解
    public ConcurrentHashMap<String, Class<?>> findClassExistAnnotation(List<Class<?>> classes) {
        for (Class<?> aClass: classes) {
            ExService exService = aClass.getDeclaredAnnotation(ExService.class);
            if (exService != null) {
                // bean的id是类名小写
                String beanId = toLowerCaseFirstOne(aClass.getSimpleName());
                beans.put(beanId, aClass);
                continue;
            }
        }
        return beans;
    }

    // 首字母转小写
    public static String toLowerCaseFirstOne(String s) {
        if (Character.isLowerCase(s.charAt(0)))
            return s;
        else
            return (new StringBuilder()).append(Character.toLowerCase(s.charAt(0))).append(s.substring(1)).toString();
    }
}
```

```java
public class Test002 {

    public static void main(String[] args) throws Exception {
        ExClassPathXmlApplicationContext applicationContext = new ExClassPathXmlApplicationContext("com.liufei.service");
        UserService userServiceImpl = (UserService) applicationContext.getBean("userServiceImpl");
        userServiceImpl.add();
    }
}
```

### 手写@Resourse注解

> 思路：
>
> 接着上面的，将类注入到容器中，然后遍历每个类中属性，修改其默认值（初始化对象）

```java
/**
 * @Auther: liufei
 * @Date: 2020/07/26/11:54 上午
 * @Description:
 */
@Target({TYPE, FIELD, METHOD})
@Retention(RUNTIME)
public @interface ExResource {
}
```

```java
public class ExClassPathXmlApplicationContext {

    // 扫包范围
    private String packageName;

    private ConcurrentHashMap<String, Object> beans = null;

    public ExClassPathXmlApplicationContext(String packageName) throws Exception {
        beans = new ConcurrentHashMap<>();
        this.packageName = packageName;
        initBeans();
        // 初始化类中的属性
        for(Map.Entry<String,Object> entry : beans.entrySet()) {
            Object value = entry.getValue();
            initFields(value);
        }
    }

    // 初始化对象
    public  void initBeans() throws Exception {
        // 1.使用java反射机制扫包，获取当前包下的所有类
        List<Class<?>> classes = ClassUtil.getClasses(packageName);
        // 2. 判断类上是否存在注入bean的注解
        ConcurrentHashMap<String, Object> classExistAnnotation = findClassExistAnnotation(classes);
        if (classExistAnnotation == null || classExistAnnotation.isEmpty()) {
            throw new Exception("改包下没有任何类的注解");
        }
    }

    public void initFields(Object object) throws Exception {
        Field[] fields = object.getClass().getDeclaredFields();
        for (Field field: fields) {
            ExResource exResource = field.getAnnotation(ExResource.class);
            if (exResource != null) {
                field.setAccessible(true);
                String beanId = field.getName();
                Object newInstall = getBean(beanId);
                if (newInstall != null) {
                    field.set(object, newInstall);
                }
            }
        }
    }

    public Object getBean(String beanId) throws Exception {
        if (StringUtils.isBlank(beanId)) {
            throw new Exception("beanId不能为空");
        }
        Object object = beans.get(beanId);
        return object;
    }

    // 判断类上是否有注入bean的注解
    public ConcurrentHashMap<String, Object> findClassExistAnnotation(List<Class<?>> classes) throws IllegalAccessException, InstantiationException {
        for (Class<?> aClass: classes) {
            ExService exService = aClass.getDeclaredAnnotation(ExService.class);
            if (exService != null) {
                // bean的id是类名小写
                String beanId = toLowerCaseFirstOne(aClass.getSimpleName());
                Object newInstance = aClass.newInstance();
                beans.put(beanId, newInstance);
                continue;
            }
        }
        return beans;
    }

    // 首字母转小写
    public static String toLowerCaseFirstOne(String s) {
        if (Character.isLowerCase(s.charAt(0)))
            return s;
        else
            return (new StringBuilder()).append(Character.toLowerCase(s.charAt(0))).append(s.substring(1)).toString();
    }


    public static void main(String[] args) throws ClassNotFoundException {
        Class<?> aClass = Class.forName("com.liufei.entity.User");
        System.out.println(aClass.getSimpleName());
    }
}
```

> ConcurrentMap存储的value变成Object，这样就可以实现单列

```java
@ExService
public class UserServiceImpl implements UserService {

    @ExResource
    private OrderService orderServiceImpl;

    @Override
    public void add() {
        orderServiceImpl.addOrder();
        System.out.println("userService中的添加方法");
    }
}
```

```java
public class Test002 {
    public static void main(String[] args) throws Exception {
        ExClassPathXmlApplicationContext applicationContext = new ExClassPathXmlApplicationContext("com.liufei");
        UserService userServiceImpl = (UserService) applicationContext.getBean("userServiceImpl");
        System.out.println(userServiceImpl);
        userServiceImpl.add();
    }
}
```

