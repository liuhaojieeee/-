# 1.Thymeleaf介绍

Thymeleaf是用来开发Web和独立环境项目的现代服务器端Java模板引擎。

Thymeleaf的主要目标是为您的开发工作流程带来优雅的自然模板 - HTML。可以在直接浏览器中正确 显示，并且可以作为静态原型，从而在开发团队中实现更强大的协作。

Spring官方支持的服务的渲染模板中，并不包含jsp。而是Thymeleaf和Freemarker等，而Thymeleaf 与SpringMVC的视图技术，及SpringBoot的自动化配置集成非常完美，几乎没有任何成本，你只用关注 Thymeleaf的语法即可。



# 2.Thymeleaf特点

1.动静结合：Thymeleaf 在有网络和无网络的环境下皆可运行，即它可以让美工在浏览器查看页面 的静态效果，也可以让程序员在服务器查看带数据的动态页面效果。这是由于它支持 html 原型， 然后在 html 标签里增加额外的属性来达到模板+数据的展示方式。浏览器解释 html 时会忽略未定 义的标签属性，所以 thymeleaf 的模板可以静态地运行；当有数据返回到页面时，Thymeleaf 标 签会动态地替换掉静态内容，使页面动态显示。 

2.开箱即用：它提供标准和spring标准两种方言，可以直接套用模板实现JSTL、 OGNL表达式效果， 避免每天套模板、改jstl、改标签的困扰。同时开发人员也可以扩展和创建自定义的方言。 

3.多方言支持：Thymeleaf 提供spring标准方言和一个与 SpringMVC 完美集成的可选模块，可以快 速的实现表单绑定、属性编辑器、国际化等功能。 

4.与SpringBoot完美整合，SpringBoot提供了Thymeleaf的默认配置，并且为Thymeleaf设置了视 图解析器，我们可以像以前操作jsp一样来操作Thymeleaf。代码几乎没有任何区别，就是在模板 语法上有区别。



# 3.环境搭建

## 3.1.创建project(thymeleaf-demo)

### pom

```java
 <!--spring boot 依赖-->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.2.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <dependencies>
        <!--thymeleaf 依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <!--web mvc 模块依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!--test 测试依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <!--maven 打包便衣依赖-->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```



### yml配置

```java
server: #端口
  port: 8083
spring: #清除页面缓存
  thymeleaf:
    prefix: file:src/main/resources/templates/
    cache: false
```



### 启动类



### 默认配置

##### 1.新建templates包结构在resouces下

##### 2.thymeleaf默认从templates读取静态页面

​	a.默认前缀： classpath:/templates/

​	b.默认后缀： .html

### TestController

```java
import com.mr.entity.Student;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.ui.ModelMap;
import org.springframework.web.bind.annotation.GetMapping;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;


@Controller
public class TestController {
    @GetMapping("hello")//定义url使用modelmap返回数据
    public String test(ModelMap map){
        map.put("name","tomcat");
        return "hello";
    }

    @GetMapping("student")
    public String student(ModelMap map){
        Student student=new Student();
        student.setCode("007");
        student.setPass("9527");
        student.setAge(18);
        student.setLikeColor("<font color='red'>红色</font>");
        map.put("stu",student);
        return "student";
    }

    @GetMapping("list")
    public String list(ModelMap map){
        Student s1=new Student("001","111",18,"red");
        Student s2=new Student("002","222",19,"red");
        Student s3=new Student("003","333",16,"blue");
        Student s4=new Student("004","444",28,"blue");
        Student s5=new Student("005","555",68,"blue");
            //转为List
            map.put("stuList", Arrays.asList(s1,s2,s3,s4,s5));
            return "list";
    }

    @GetMapping("each")
    public String each(Model model){

        List<Student> students = new ArrayList<>();

        for (int i = 0; i < 10; i++) {
            students.add(new Student("1"+i, "a"+ i, 15 +i, "red" + 1));
        }
        model.addAttribute("students",students);
        return "each";
    }



}
```

# 4.thymeleaf语法

##### Thymeleaf通过 ${} 来获取model中的变量，注意这不是el表达式，而是ognl表达式，但是语法非常像。

#### 1. th:text 取值输出:

##### 	th:text 中的thymeleaf并不会被认为是变量，而是一个字符串

##### 	数字不需要任何特殊语法， 写的什么就是什么，而且可以直接进行算术运算

##### 	但是要注意， th 指令是依赖于h5的 如果浏览器不支持h5 那么可以使用 data-th-text 指令 

##### 	如果值中包含html代码需要渲染的 则使用 th:utext 来输出. 刚才获取变量值，我们使用的是经典的 对象.属性	名 方式。但有些情况下，我们的属性名可能本身也是变量，

##### 	${stu.age} 可以写作 ${stu['age']}

#### 2.自定义变量:

##### 	首先在 div 上 用 th:object="${stu}" 获取stu的值，并且保存 

##### 	在 div 内部的任意元素上，可以通过 *{属性名} 的方式，来获取stu中的属性，这样就省去 了大量的 stu. 前缀	了

#### 3.方法

##### 例子:

##### 	th:text="${stu.code.substring(2)}"

##### 	th:text="${stu.code.split('')[1]}"

##### 可以调用字符串截取，分割等函数

##### Thymeleaf中提供了一些内置对象，并且在这些对象中提供了一些方法，方便我们来调用。获取这些对 象，需要使用    #对象名     来引用。

​	![image-20210311223025364](C:\Users\15456\AppData\Roaming\Typora\typora-user-images\image-20210311223025364.png)

# 5.创建模板

#### 1.

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>测试thymeleaf</title>
</head>
<body >
        你好，<span th:text="${name}"></span>
</body>
</html>
```

#### 2.

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>循环</title>
</head>
<body>
    <ul>
        <li>编码--密码--年龄</li>
        <li th:each="stu : ${stuList}">
            <span th:text="${stu.code}"></span>
            <span th:text="${stu.pass}"></span>
            <span th:text="${stu.age}"></span>
        </li>
    </ul>
    <ul>
        <li >下标--编码--密码--年龄</li>
        <li th:each="stu,stat : ${stuList}">
            <span th:text="${stat.index}"></span>--
            <span th:text="${stu.code}"></span>--
            <span th:text="${stu.pass}"></span>--
            <span th:text="${stu.age}"></span>
            <span th:if="${stu.age>60}">可以退休</span>
            <span th:unless="${stu.age>60}">不能退休</span>
            <span th:switch="${stu.code}">
            <span th:case="'001'">1号员工，骨灰级</span>
            <span th:case="'002'">2号员工，元老级</span>
            <span th:case="'003'">3号员工，老员工</span>
            <span th:case="'004'">4号员工，新员工</span>
            <span th:case="*">临时工</span>
            </span>
            </li>
    </ul>

    <script th:inline="javascript">
        //预处理js值
        const stuList=/*[[${stuList}]]*/;
        const stu=/*[[${stuList[0]}]]*/;
        const age=/*[[${stuList[0].age}]]*/;
        console.log("集合："+stuList);
        console.log("对象："+stu);
        console.log("属性："+age);
        stuList.forEach(stu=>{
            console.log(stu.code+" "+stu.pass)
        })
    </script>
</body>
</html>
```



### 3.

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>student</title>
</head>
<body>
    你好<span th:text="${stu.code}"></span><br>
    密码是: <span data-th-text="${stu.pass}"></span><br>
    您的年龄是：<span th:text="${stu['age']}"></span><br>
    您喜欢的颜色是<span th:utext="${stu.likeColor}"></span>

    <hr/>
    <div th:object="${stu}">
        <span th:text="*{code}"></span><br>
        <span th:text="*{pass}"></span><br>
        <span th:text="*{age}"></span><br>
        <span th:utext="*{likeColor}"></span>
    </div>

    <span th:text="${stu.code.substring(2)}"></span><br>
    <span th:text="${stu.code.split('')[2]}"></span><br>

    <span th:text="${#dates}"></span>
    <hr>
    <span th:text="'666'"></span>
    <hr>
    <span th:if="${stu.age < 18}">
    成年
    </span>
    <hr>
    <span th:text="'你好' +${stu.code}+ ' 666,你好牛逼啊!'"></span><br>
    <span th:text="${stu.age} *2"></span><br>
    <span th:text="${stu.age} >= 18 ? '成年' : '未成年'"></span><br>
   
</body>
</html>
```



