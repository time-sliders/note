1. 不要使用 `double` 构造 `BigDecimal`（推荐使用`String`）

   > 正例：`new BigDecimal("2.01");`
   > 反例：`new BigDecimal(2.01);`

2. 数据比较

   BigDecimal` 不要使用 `equals` 以及 `==` 做比较，请使用 `compare` 方法比较。

   > *BigDecimal 1.0 与1.00 在equals 比较时，会因为精度 scale 不同，返回 false*。

3. 优先使用 `BigDecimal.ZERO` 等 `BigDecimal` 已经提供过的常量，而不是自己 `new` 一个。

4. 在页面数据汇总展示时，需要考虑是先取精度在累加，还是先累加在取精度，需要保证同一项数据在不同页面的取值逻辑保持一致，避免不同的页面差几分钱的一些情况。

5. 除法计算精度

  ```java
  BigDecimal b = new BigDecimal("10");
  BigDecimal e = new BigDecimal("7");
  
  // 未指定精度，则以第一个数的精度来做为结果精度
  b.divide(e, BigDecimal.ROUND_HALF_UP));										// 1
  
  // 只指定第二个数的精度，并不会影响结果精度
  b.divide(e.setScale(2,BigDecimal.ROUND_HALF_UP), BigDecimal.ROUND_HALF_UP) // 1
  // 当设置第一个数的精度时，结果精度随之改变，由此可见：如果 divide 方法如果未指定 scale ，则需要在除之前预设精度
  b.setScale(2,BigDecimal.ROUND_HALF_UP).divide(e, BigDecimal.ROUND_HALF_UP);// 1.43
  // 直接指定计算精度
  b.divide(e, 2,BigDecimal.ROUND_HALF_UP));									// 1.43
  
  
  ```

  我们分析 BigDecimal 的源码可以看到 如果算法未指定精度，默认使用的是调用 divide 方法的对象的 scale 属性。

  ```java
  public BigDecimal divide(BigDecimal divisor, int roundingMode) {
      return this.divide(divisor, scale/*这里的 scale 是 this.scale*/, roundingMode);
  }
  ```

6. 计算优先级

  **加法、减法、乘法    >  除法**

  将除法运算尽量靠后，减少丢失的精度。

7. 