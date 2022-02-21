## 对象的通用方法
#### equals
在Java中，==表示比较两个对象是否是同一个实例，而equals方法用于比较逻辑相等，equals方法需要符合：
1. 反身性：对于任何非空的参考值 x，x.equals(x) 必须返回 true。
2. 对称性：对于任何非空参考值 x 和 y，x.equals(y) 必须在且仅当 y.equals(x) 返回 true 时返回 true。
3. 传递性：对于任何非空的引用值 x, y, z，如果 x.equals(y) 返回 true，y.equals(z) 返回 true，那么 x.equals(z) 必须返回 true。
4. 一致性：对于任何非空的引用值 x 和 y, x.equals(y) 的多次调用必须一致地返回 true 或一致地返回 false，前提是不修改 equals 中使用的信息。

#### hashCode
在覆盖 equals 的类中，必须覆盖 hashCode。否则无法在HashMap、HashSet等集合中正常使用。需符合约定：
1. 当在应用程序执行期间重复调用对象的 hashCode 方法时，它必须一致地返回相同的值，前提是不对 equals 比较中使用的信息进行修改。这个值不需要在应用程序的不同执行之间保持一致。
2. 如果根据 equals(Object) 方法判断出两个对象是相等的，那么在两个对象上调用 hashCode 必须产生相同的整数结果。
3. 如果根据 equals(Object) 方法判断出两个对象不相等，则不需要在每个对象上调用 hashCode 时必须产生不同的结果。但是，程序员应该知道，为不相等的对象生成不同的结果可能会提高 hash 表的性能。

> 有了equals为什么还要hashCode？hashCode返回一个整数，可以作为一个标识（当然，需要考虑哈希碰撞）。

#### 计算hashCode的常规方式
> 重要字段：会影响equals的字段

1. 声明result变量，初始化为第一个重要字段的hashCode，假设为c。
2. 对象中剩余的重要字段f，执行以下操作：
   * 如果是基本数据类型，计算 Type.hashCode(f)，其中 type 是与 f 类型对应的包装类
   * 如果是引用类型，则递归调用hashCode；如果字段的值为null，则使用0
   * 如果是数组，则要调用每个元素的hashCode，可参照 Arrays.hashCode
   * 将以上步骤计算的hashCode合并到结果
   ```
   result = 31 * result + c;
   ```


例如：
```
@Override
public int hashCode() {
    int result = Short.hashCode(field1);
    result = 31 * result + field2.hashCode();
    result = 31 * result + field3.hashCode();
    return result;
}
```