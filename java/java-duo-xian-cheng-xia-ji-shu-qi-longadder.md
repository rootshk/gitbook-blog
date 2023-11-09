# \[Java] 多线程下计数器 LongAdder

#### 服用方式

```
public class Test {

    // 计数器
    public static LongAdder a = new LongAdder();

    // 异步方法
    public void addTest() {
        a.inrcement();// 等于a.add(1L);
    }

    // 异步方法
    public void subtractionTest() {
        a.decrement();// 等于a.add(-1L);
    }

    // 获取
    public Long getSum() {
       return a.sum();
    }

}
```
