---
title: Java8新写法
date: 2018-01-22 15:09:10
categories:
tags: ['lambda', '语法糖']
---
Jdk8 lambd新写法
<!-- more -->
### Map.merge
```java
public static void main(String[] args) throws InterruptedException {
    Map<String, Integer> pageVisits = new HashMap<>();
    String page = "https://agiledeveloper.com";
    incrementPageVisit(pageVisits, page);
    incrementPageVisit(pageVisits, page);
    incrementPageVisitNew(pageVisits, page);
    System.out.println(pageVisits);
}

/**
 * 统计Map中key出现的次数
 *
 * @param pageVisits
 * @param page
 */
public static void incrementPageVisit(Map<String, Integer> pageVisits, String page) {
    if (!pageVisits.containsKey(page)) {
        pageVisits.put(page, 0);
    }
    pageVisits.put(page, pageVisits.get(page) + 1);
}

/**
 * 改进后写法
 * @param pageVisits
 * @param page
 */
public static void incrementPageVisitNew(Map<String, Integer> pageVisits, String page) {
    /*
     * key：map传入的key
     * value：不存在的时候默认值不能为null
     * remappingFunction：表达式，查询计算值的表达式
     */
    pageVisits.merge(page, 1, (oldValue, value) -> oldValue + value);
}
```
### List.Stream
```java
class Car{
    private String make;
    private String model;
    private int year;

    public Car(String make, String model, int year) {
        this.make = make;
        this.model = model;
        this.year = year;
    }

    public String getMake() {
        return make;
    }

    public String getModel() {
        return model;
    }

    public int getYear() {
        return year;
    }
}

public class Main {
    public static void main(String[] args) throws InterruptedException {
        List<Car> cars = Arrays.asList(
                new Car("Jeep", "Wrangler", 2011),
                new Car("Jeep", "Comanche", 1990),
                new Car("Dodge", "Avenger", 2010),
                new Car("Buick", "Cascada", 2016),
                new Car("Ford", "Focus", 2012),
                new Car("Chevrolet", "Geo Metro", 1992)
        );
        System.out.println(getModelsAfter2000UsingFor(cars));
        System.out.println(getModelsAfter2000UsingForNew(cars));
    }

    /**
     * 获取2000年制造的汽车的名称，然后按年份进行排序
     * @param cars
     * @return
     */
    public static List<String> getModelsAfter2000UsingFor(List<Car> cars){
        //过滤需要的数据，在数据量特别大时，先过滤后排序较快
        List<Car> sortCar = new ArrayList<>();
        for (Car car : cars) {
            if (car.getYear() > 2000){
                sortCar.add(car);
            }
        }
        //通过实践排序
        sortCar.sort(new Comparator<Car>() {
            @Override
            public int compare(Car o1, Car o2) {
                return Integer.valueOf(o1.getYear()).compareTo(o2.getYear());
            }
        });
        List<String> models = new ArrayList<>();
        for (Car car : sortCar) {
            models.add(car.getModel());
        }
        return models;
    }
    public static List<String> getModelsAfter2000UsingForNew(List<Car> cars){
        /*
         * filter：lambda过滤数据
         * sorted：数据排序
         * map：返回有给定的参数结果的流
         * collect：收集流的数据
         */
        return cars.stream()
                .filter(car -> car.getYear() > 2000)
                .sorted(Comparator.comparing(Car::getYear))
                .map(Car::getModel)
                .collect(Collectors.toList());
    }


}
```

### for循环
循环自定次数
```java
/**
 * 循环指定次数
 * @param num
 */
public static void fori(int num) {
    for (int i = 0; i < num; i++) {
        System.out.println(i);
    }
}

public static void foriNew(int num) {
    IntStream.range(0, num).forEach(System.out::println);
}
```
循环次数包含当前值
```java
/**
 * 循环指定次数，包含当前值
 * @param num
 */
public static void fori(int num) {
    for (int i = 0; i <= num; i++) {
        System.out.println(i);
    }
}

public static void foriNew(int num) {
    IntStream.rangeClosed(0, num).forEach(System.out::println);
}
```

跳着循环
```java
/**
 * 循环指定次数，包含当前值
 *
 * @param num
 */
public static void fori(int num) {
    for (int i = 0; i <= num; i += 3) {
        System.out.printf("%d \t", i);
    }
}

public static void foriNew(int num) {
    /*
     * iterate：后面lambda表示通过前面的值计算出后一个新值
     * 这里需要计算循环次数，不方便，可以采用jdk9中takeWhile
     * 逆向循环用减法
     */
    IntStream.iterate(0, e -> e + 3).limit(num/3 + 1).forEach(d -> System.out.printf("%d \t", d));
    System.out.println();
    IntStream.rangeClosed(0, num)
            .filter(e -> e%3 == 0)
            .forEach(d -> System.out.printf("%d \t", d));
}
```
### 函数式接口
可以使用lambda表达式
要求：
* 接口
* 只有一个抽象方法，默认方法，静态方法不算

默认情况下只要一个抽象方法的接口都会默认为函数式，建议写**FunctionalInterface**注解，避免后期修改导致出错
如下：
```java
@FunctionalInterface
interface Transformer<T>{
    void transform(T input);
}

public class Main {
    public static void main(String[] args) throws InterruptedException {
        Transformer<Stream<String>> transformer = System.out::println;
    }
}
```

### lambda表达式对象
Function：带返回指的表达式
```java
Function<Integer, Integer> function = (Integer num) -> {
    if (num == null){
        return 0;
    }
    return num / 5;
};
System.out.println(function.apply(10));
```
Consumer：不带返回值的表达式
```java
Transformer<Integer> transformer = (Integer num) -> {
    if (num != null) {
        System.out.println(num / 5);
    }
};
transformer.transform(10);
```
Predicate：表示输入是否符合条件，用于filter
```java
Predicate<Integer> predicate = (Integer num) -> num != null && num % 5 == 0;
System.out.println(predicate.test(10));
```
