# java _通过反射获取被注解的字段的文字描述

应用场景：需要字段和文字做匹配时可用：

1.定义注解======================================================

```
@Target({ElementType.PARAMETER,ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ExcelTitle {

     String title();
}
```

2.使用注解=====================================================

```
@Data
public class OrderExcel {


    @ExcelTitle(title = "名称")
    private String title;

    @ExcelTitle(title = "所属领域")
    private String areaNames;
}
```

2.通过反射拿到属性===============================================

 

```
public class Test {

    public static void main(String[] args) {
        // 获取类对象
        Class<OrderExcel> orderExcelClass = OrderExcel.class;
        // 获取类属性
        Field[] declaredFields = orderExcelClass.getDeclaredFields();
        for (Field f : declaredFields) {
            if (f.isAnnotationPresent(ExcelTitle.class)) {
                // 获取注解值
                System.out.print(f.getAnnotation(ExcelTitle.class).title() + ",");
            }


        }
    }
}
```