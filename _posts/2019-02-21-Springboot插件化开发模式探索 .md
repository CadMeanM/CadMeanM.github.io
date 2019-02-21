---
layout:     post
title:      Springboot插件化开发
subtitle:   开发模式探索
date:       2019-02-21
author:     Cadmean
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 分布式
---

### 思考
在网上看了几篇插件化开发的文章，都比较繁琐，不直接。我理解最关键的部分是如何在springboot启动后，加载外部的依赖jar包，还有从如何从配置中得到实现类名并加载。

通过这样的模式，可以实现主程序中只有一个接口，通过依赖lib+配置的方式在主程序中调起实际的实现方法并执行。

### Serviceloader方式
serviceloader是java提供的spi模式的实现。按照接口开发实现类，而后配置，java通过ServiceLoader来实现统一接口不同实现的依次调用。与springboot结合后，该方式要实现的关键点有：
1. 在springboot启动时调用加载lib目录下所有jar包的方法。
2. 在需要调用服务时，使用ServiceLoader的方式来动态加载所有类，只需要定义好接口就行了。
3. 定义接口和实现类，接口必须在主程序中加载，实现类可以不在主程序中。

上代码，这里没有在springboot启动时加载lib目录，而是为了测试方便在每次调用接口时都加载一次。
```
//接口类
package com.example.demo;
public interface IHello {
    public void sayHello();
}
```

```
//实现类，这个类放在主程序中
public class dogHello implements IHello {
    @Override
    public void sayHello() {
        System.out.println("WANG WANG");
    }
}
```

```
//实现类，放在catHello.jar中
package com.example.demo;

public class catHello implements IHello {
    @Override
    public void sayHello() {
        System.out.println("MIAO MIAO");
    }
}

```

```
//META-INF/services/com.example.demo.IHello文件
com.example.demo.dogHello
com.example.demo.catHello
```

```
//动态加载lib下所有jar包的方法
public class Utils {
    public static String getApplicationFolder() {
        String path = Utils.class.getProtectionDomain().getCodeSource().getLocation().getPath();
        return new File(path).getParent();
    }

    public static void loadJarsFromAppFolder(String sub_folder) throws Exception {
        String path = "./" + sub_folder;
        System.out.println(path);
        File f = new File(path);
        if (f.isDirectory()) {
            for (File subf : f.listFiles()) {
                if (subf.isFile()) {
                    System.out.println(subf.getName());
                    loadJarFile(subf);
                }
            }
        } else {
            loadJarFile(f);
        }
    }

    public static void loadJarFile(File path) throws Exception {
        URL url = path.toURI().toURL();
        URLClassLoader classLoader = (URLClassLoader) ClassLoader.getSystemClassLoader();
        System.out.println(classLoader.getClass());
        Method method = URLClassLoader.class.getDeclaredMethod("addURL", URL.class);
        method.setAccessible(true);
        method.invoke(classLoader, url);
    }
}
```

```
//主程序

@SpringBootApplication
@Controller
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @RequestMapping(value = "/",method = RequestMethod.GET)
    @ResponseBody
    public String index() throws Exception {
        Utils.loadJarsFromAppFolder("lib");
        ServiceLoader<IHello> serviceLoader= ServiceLoader.load(IHello.class);
        for (IHello hello:serviceLoader){
            hello.sayHello();
        }
        return "hello world!";
    }

}

```

只需要把catHello.jar放在编译后代码的lib目录下，然后执行，就可以实现访问一次localhost:8080，就在控制台输出一次
```
WANG WANG
MIAO MIAO
```

### 自定义方式
基于上面的方式，Serviceloader其实是有缺陷的，就是必须在META-INF里定义接口名称的文件，在文件中才能写上实现类的类名，如果一个项目里插件化的东西比较多，那很可能会出现越来越多配置文件的情况。

考虑把配置全部都放到application.yml里。

```
server :
  port : 8080
impl:
  name : com.example.demo.IHello
  clazz :
    - com.example.demo.dogHello
    - com.example.demo.catHello
```

接下来定义一下读取impl配置的类，这里用了lombok
```
package com.example.demo.configuration;

import lombok.Getter;
import lombok.Setter;
import lombok.ToString;
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("impl")
@ToString
public class ClassImpl {
    @Getter
    @Setter
    String name;

    @Getter
    @Setter
    String[] clazz;
}

```
主程序里的代码改成：
```
package com.example.demo;

@SpringBootApplication
@Controller
@EnableConfigurationProperties({ClassImpl.class})
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @Autowired
    ClassImpl classImpl;

    @RequestMapping(value = "/",method = RequestMethod.GET)
    @ResponseBody
    public String index() throws Exception {
        Utils.loadJarsFromAppFolder("lib");
        for (int i=0;i<classImpl.getClazz().length;i++) {
            Class helloClass= Class.forName(classImpl.getClazz()[i]);
            IHello hello = (IHello) helloClass.newInstance();
            hello.sayHello();
        }
        return "hello world!";
    }

}

```
当然还有很多异常处理没有做，实际使用中要记得补上。还有就是java的反射机制效率并不高，所以要考虑反射出实例一次后，尽量尝试复用，这就要求接口设计要好一点了。

其实根据这样的模式，其实都可以实现在数据库中配置实现类了。