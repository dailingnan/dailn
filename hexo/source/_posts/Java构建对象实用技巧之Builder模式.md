---
title: Java构建对象实用技巧之Builder模式
date: 2019-01-05 17:13:43
categories: "设计模式"
tags: [设计模式,Java]
---

<Excerpt in index | 首页摘要> 

# 背景

在实际中，我们经常遇到这种场景。在实例化一些JavaBean的时候，需要给很多属性赋值。我们通常的做法是给必要的属性一个一个通过set方法赋值。

<!-- more -->
<The rest of contents | 余下全文>

# 实践

例如

有这样一个用户对象

```java
public class UserBean {
    private String name;
    private int age;
    private String sex;
 
    public String getName() {
        return name;
    }
 
    public void setName(String name) {
        this.name = name;
    }
 
    public int getAge() {
        return age;
    }
 
    public void setAge(int age) {
        this.age = age;
    }
 
    public String getSex() {
        return sex;
    }
 
    public void setSex(String sex) {
        this.sex = sex;
    }
 
    @Override
    public String toString() {
        return "UserBean{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", sex='" + sex + '\'' +
                '}';
    }
}
```

接下来我们实例化

```java
public class Test {
 
    public static void main(String[] args) {
        UserBean userBean = new UserBean();
        userBean.setName("戴岭南");
        userBean.setAge(23);
        userBean.setSex("男");
       System.out.println( userBean.toString());
    }
}
```

```java
UserBean{name='戴岭南', age=23, sex='男'}
```

恩，这样不是很正常吗，完全没有任何的不适当。

接下来我们增加一些其他相关参数

```java
 
public class UserBean {
    private String name;
    private int age;
    private String sex;
    private String idCard;
    private int weight;
    private int height;
    private String address;
    private String education;
    private String birthday;
    private String post;
    private String marriage;
    private String degree;
 
    public String getName() {
        return name;
    }
 
    public void setName(String name) {
        this.name = name;
    }
 
    public int getAge() {
        return age;
    }
 
    public void setAge(int age) {
        this.age = age;
    }
 
    public String getSex() {
        return sex;
    }
 
    public void setSex(String sex) {
        this.sex = sex;
    }
 
    public String getIdCard() {
        return idCard;
    }
 
    public void setIdCard(String idCard) {
        this.idCard = idCard;
    }
 
    public int getWeight() {
        return weight;
    }
 
    public void setWeight(int weight) {
        this.weight = weight;
    }
 
    public int getHeight() {
        return height;
    }
 
    public void setHeight(int height) {
        this.height = height;
    }
 
    public String getAddress() {
        return address;
    }
 
    public void setAddress(String address) {
        this.address = address;
    }
 
    public String getEducation() {
        return education;
    }
 
    public void setEducation(String education) {
        this.education = education;
    }
 
    public String getBirthday() {
        return birthday;
    }
 
    public void setBirthday(String birthday) {
        this.birthday = birthday;
    }
 
    public String getPost() {
        return post;
    }
 
    public void setPost(String post) {
        this.post = post;
    }
 
    public String getMarriage() {
        return marriage;
    }
 
    public void setMarriage(String marriage) {
        this.marriage = marriage;
    }
 
    public String getDegree() {
        return degree;
    }
 
    public void setDegree(String degree) {
        this.degree = degree;
    }
 
    @Override
    public String toString() {
        return "UserBean{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", sex='" + sex + '\'' +
                ", idCard='" + idCard + '\'' +
                ", weight=" + weight +
                ", height=" + height +
                ", address='" + address + '\'' +
                ", education='" + education + '\'' +
                ", birthday='" + birthday + '\'' +
                ", post='" + post + '\'' +
                ", marriage='" + marriage + '\'' +
                ", degree='" + degree + '\'' +
                '}';
    }
}
```

继续按照set方式对对象进行实例化

```java
public class Test {
 
    public static void main(String[] args) {
        UserBean userBean = new UserBean();
        userBean.setName("戴岭南");
        userBean.setAge(23);
        userBean.setSex("男");
        userBean.setAddress("湖南省岳阳市");
        userBean.setBirthday("19950827");
        userBean.setEducation("蓝翔毕业");
        userBean.setDegree("学士");
        userBean.setHeight(175);
        userBean.setIdCard("430626***");
        userBean.setMarriage("未婚");
        userBean.setPost("Java攻城狮");
        userBean.setWeight(120);
       System.out.println( userBean.toString());
    }
}
```

```java
UserBean{name='戴岭南', age=23, sex='男', idCard='430626***', weight=120, height=175, address='湖南省岳阳市', education='蓝翔毕业', birthday='19950827', post='Java攻城狮', marriage='未婚', degree='学士'}
```

恩，现在看是不是有点。。。。，实例化一个对象就这么大一段代码了。十分的不美观，这里也可以采用构造函数实例化，不过构造函数对参数限制性大，灵活性不高。假如每一个参数都不是必须的，那么我们就要重载对应参数个数个构造函数，更加不灵活。

接下来我们采用Builder模式来改造一下

```java
package com.dailingnan.build;
 
/**
 * @author dailn
 * @Classname UserBean
 * @Desc
 * @create 2018-12-15 9:41
 **/
public class UserBean {
    private String name;
    private int age;
    private String sex;
    private String idCard;
    private int weight;
    private int height;
    private String address;
    private String education;
    private String birthday;
    private String post;
    private String marriage;
    private String degree;
 
 
 
    public static  class  Build{
        private String name;
        private int age;
        private String sex;
        private String idCard;
        private int weight;
        private int height;
        private String address;
        private String education;
        private String birthday;
        private String post;
        private String marriage;
        private String degree;
 
        public Build  name(String name){
                this.name=name;
                return this;
        }
        public Build  sex(String sex){
            this.sex=sex;
            return this;
        }
        public Build  age(int age){
            this.age=age;
            return this;
        }
        public Build  idCard(String idCard){
            this.idCard=idCard;
            return this;
        }
        public Build  weight(int weight){
            this.weight=weight;
            return this;
        }
        public Build  height(int height){
            this.height=height;
            return this;
        }
        public Build  address(String address){
            this.address=address;
            return this;
        }
        public Build  education(String education){
            this.education=education;
            return this;
        }
        public Build  post(String post){
            this.post=post;
            return this;
        }
        public Build  marriage(String marriage){
            this.marriage=marriage;
            return this;
        }
        public Build  birthday(String birthday){
            this.birthday=birthday;
            return this;
        }
        public Build  degree(String degree){
            this.degree=degree;
            return this;
        }
 
        public UserBean build(){
            return new UserBean(this);
        }
 
    }
 
    private UserBean(Build build) {
        this.name=build.name;
        this.age=build.age;
        this.sex=build.sex;
        this.idCard=build.idCard;
        this.weight=build.weight;
        this.height=build.height;
        this.address=build.address;
        this.education=build.education;
        this.birthday=build.birthday;
        this.post=build.post;
        this.marriage=build.marriage;
        this.degree=build.degree;
    }
 
 
 
    @Override
    public String toString() {
        return "UserBean{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", sex='" + sex + '\'' +
                ", idCard='" + idCard + '\'' +
                ", weight=" + weight +
                ", height=" + height +
                ", address='" + address + '\'' +
                ", education='" + education + '\'' +
                ", birthday='" + birthday + '\'' +
                ", post='" + post + '\'' +
                ", marriage='" + marriage + '\'' +
                ", degree='" + degree + '\'' +
                '}';
    }
}
```

采用Builder构建对象

```java
public class Test {
 
    public static void main(String[] args) {
        UserBean userBean =new UserBean.Build().name("戴岭南").age(23).sex("男").post("Java攻城狮")
                .address("湖南省岳阳市").education("蓝翔毕业").birthday("19950827").weight(120)
                .degree("学士").height(175).idCard("430626***").marriage("未婚").build();
       System.out.println( userBean.toString());
    }
}
```

```java
UserBean{name='戴岭南', age=23, sex='男', idCard='430626***', weight=120, height=175, address='湖南省岳阳市', education='蓝翔毕业', birthday='19950827', post='Java攻城狮', marriage='未婚', degree='学士'}
```

这种方式创建对象是不是很简捷。

# 总结

​	Builder模式的确有它自身的不足。为了创建对象，必须先创建它的构建器。虽然创建构建器的开销在实践中可能不那么明显，但是在某些十分注重性能的情况下，可能就成问题了。Builder模式还比重叠构造器模式更加冗长，因此它只有在很多参数的时候才使用，比如四个或者更多个参数。

补充：后期通过lombok插件只需要一个注解就可实现build模式哦！

参考书籍《effective in  java》