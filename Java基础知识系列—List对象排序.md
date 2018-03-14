# Java基础知识系列—List对象排序

## Collectins工具类如下提供排序方法：

```java
public static <T extends Comparable<? super T>> void sort(List<T> list) {
    list.sort(null);
}


public static <T> void sort(List<T> list, Comparator<? super T> c) {
    list.sort(c);
}
```

- 使用方法一时对象T必需实现Comparable接口否则编译不通过，排序时使用Comparable的compareTo(T o)进行元素大小比较；

- 方法二需要显式指定比较器（实现了Comparator接口），排序时使用指定比较器的compare(T o1, T o2)方法进行元素大小比较；

## 自定义对象排序的例子

```java
public class ObjectSort {

    public static void main(String[] args) {
        List<Student> students = new ArrayList<>();
        students.add(new Student("0022", "zhangsan"));
        students.add(new Student("0001", "lisi"));
        students.add(new Student("0012", "xiaozhang"));
        students.add(new Student("0002", "xiaozhao"));
        Collections.sort(students);
        for (Student s : students) {
            System.out.println(s.getNo() + "," + s.getName());
        }
    }
}

class StudentComparator implements Comparator<Student> {

    @Override
    public int compare(Student o1, Student o2) {
        Integer no1 = Integer.valueOf(o1.getNo());
        Integer no2 = Integer.valueOf(o2.getNo());
        return no1 > no2 ? 1 : (no1.equals(no2)) ? 0 : -1;
    }
}

class Student implements Comparable<Student> {

    private String no;

    private String name;

    public String getNo() {
        return no;
    }

    public void setNo(String no) {
        this.no = no;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Student(String no, String name) {
        this.no = no;
        this.name = name;
    }

    @Override
    public int compareTo(Student o) {
        return new StudentComparator().compare(this, o);
    }
}
```

根据运行结果可以发现，默认是按照升序进行排序的。而在实际应用中经常是按照倒序排序，该如何解决呢？

- 调用Collections.reverse(List<?> list)方法将升序排序结果进行倒转，但效率不高;

- 修改StudentComparator类的compare方法，代码如下：

```java
class StudentComparator implements Comparator<Student> {

    @Override
    public int compare(Student o1, Student o2) {
        Integer no1 = Integer.valueOf(o1.getNo());
        Integer no2 = Integer.valueOf(o2.getNo());
        // 取反
        return -(no1 > no2 ? 1 : (no1.equals(no2)) ? 0 : -1);
    }
}
```

## 总结

1、推荐使用显式指定比较器方法，Comparable的compare方法已实现或防止compare方法被修改的可能；

2、Collectins工具类的sort方法的过程是首先将List转换成数组，然后选择排序方法对数组进行排序。

