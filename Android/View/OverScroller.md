View的overScrollBy，可能能用于iOS的回弹效果

似乎提供了对超出滑动边界的情况的处理

----------------

springBack和类似startScroll，只是传参的应用场景不同。springBack用于滚动到一个合法范围，startScroll用于滚到自行设定的距离。

-----------------

有一个fling重载方法为
```
fling(int startX, int startY, int velocityX, int velocityY,
            int minX, int maxX, int minY, int maxY, int overX, int overY)
```
最后面两个参数，表示可以超过的范围，但最终还是要滚回到min或者max，**触底反弹**（应该是滑动最顶部或最底部，然后再滑动一点距离，最后回到最顶部或者最底部）