```java
public interface Comparator<T>{
    int compare(T o1, T o2);
    ...
}
```
该接口描述比较的参数权重情况，最终结果为按权重值由小到大排列  
compare方法大于0，就把前一个数和后一个数交换，如果小于等于0就保持原位置，不进行交换
