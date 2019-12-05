---
layout: post
title:  "Java String字符串的一些简单测试"
date:   2019-12-05 21:15
categories: 性能
tags: 编程 Java 
---
* content
{:toc}

一个小小的测试, 相关代码如下：

1. 待测试的类行为


```java
@Data
@Accessors(chain = true)
public class CustomerRes {

    private String customerId;
    private String name;
    private String identityNumber;

    public String toStringJoinWithPlus(){
        String str =  "CustomerRes[";
        if (customerId != null) str += " customerId=" + customerId;
        if (name != null) str += ", name=" + name;
        if (identityNumber != null) str += ", identityNumber=" + identityNumber;
        str +="]";
        return str;
    }

    public String toStringJoinWithBuilder(){
        StringBuilder strBder = new StringBuilder("CustomerRes[");
        if (customerId != null) strBder.append(" customerId=").append(customerId);
        if (name != null) strBder.append(", name=").append(name);
        if (identityNumber != null) strBder.append(", identityNumber=").append(identityNumber);
        strBder.append("]");
        return strBder.toString();
    }

    public String toStringJoinWithPlusMask(){
        String str =  "CustomerRes[";
        if (customerId != null) str += " customerId=" + customerId;
        if (name != null) str += ", name=" + DisplayUtil.displayName(name);
        if (identityNumber != null) str += ", identityNumber=" + DisplayUtil.displayIdentityNumber(identityNumber);
        str +="]";
        return str;
    }

    public String toStringJoinWithBuilderMask(){
        StringBuilder strBder = new StringBuilder("CustomerRes[");
        if (customerId != null) strBder.append(" customerId=").append(customerId);
        if (name != null) strBder.append(", name=").append(DisplayUtil.displayName(name));
        if (identityNumber != null) strBder.append(", identityNumber=").append(DisplayUtil.displayIdentityNumber(identityNumber));
        strBder.append("]");
        return strBder.toString();
    }


    public static class DisplayUtil{
        public static String displayName(String name){
            if (name == null || name.length() <= 1){
                return name;
            }

            StringBuilder strbder = new StringBuilder();
            for (int i=0; i<name.length(); i++){
                if (i==0){
                    strbder.append(name.substring(i, i+1));
                } else {
                    strbder.append("*");
                }
            }
            return strbder.toString();
        }

        public static String displayIdentityNumber(String identityNumber){
            if (identityNumber == null || identityNumber.length() <= 4){
                return identityNumber;
            }

            StringBuilder strBder = new StringBuilder();
            for (int i=0; i<identityNumber.length(); i++){
                if(i < 4){
                    strBder.append(identityNumber.substring(i, i+1));
                } else {
                    strBder.append("*");
                }
            }
            return strBder.toString();
        }
    }

}
```

2. 相关测试代码

```java
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@BenchmarkMode(Mode.AverageTime)
public class StringJointTest {

    public static void main(String[] args) throws RunnerException {
        Options options = new OptionsBuilder()
                .include(StringJointTest.class.getSimpleName())
                .forks(1)
                .build();
        new Runner(options).run();
    }


    static CustomerRes customerRes = new CustomerRes()
            .setCustomerId("10086")
            .setName("诸葛亮")
            .setIdentityNumber("3334333433343334");

    @Benchmark
    public void toStringJoinWithPlus(){
        customerRes.toStringJoinWithPlus();
    }

    @Benchmark
    public void toStringJoinWithBuilder(){
        customerRes.toStringJoinWithBuilder();
    }

    @Benchmark
    public void toStringJoinWithPlusMask(){
        customerRes.toStringJoinWithPlusMask();
    }

    @Benchmark
    public void toStringJoinWithBuilderMask(){
        customerRes.toStringJoinWithBuilderMask();
    }
}
```

3. 测试结果

每个方法执行5次迭代，每次迭代持续10分钟。单线程，计时单位为ns。

```
Benchmark                                           Mode  Cnt    Score    Error  Units
String.StringJointTest.toStringJoinWithBuilder      avgt    5  185.149 ± 12.267  ns/op
String.StringJointTest.toStringJoinWithBuilderMask  avgt    5  571.026 ± 17.660  ns/op
String.StringJointTest.toStringJoinWithPlus         avgt    5   90.799 ± 14.327  ns/op
String.StringJointTest.toStringJoinWithPlusMask     avgt    5  670.779 ± 26.607  ns/op
```

可以看到，若无方法调用，则编译器可优化 ‘+’ 拼接。
若有方法调用，StringBuilder快于 ‘+’