# 一类避免服务端大量定时器的思路

例如下面一个例子：

+ 逻辑上，每个用户都有一个 energy 属性，初始是最大容量（每个人的最大容量可能不一样）
+ 用户会使用 energy 来完成一些其它操作（不同操作需要的 energy 可能不一样）
+ 用户也会偶尔获得一些 energy，但是不会超过最大容量
+ 如果 energy 不是最大容量，energy 会随着时间逐渐线性增长（每个人的增长速率可能不一样）
+ energy 增长到最大容量后就不再增长

直接的解决方案是，为每个 energy 没有达到最大容量的用户配备一个定时器，定时增加 energy，直到达到最大容量。

上面这个方案，在活跃用户很多时，会需要大量的定时器，也有大量的数据操作，很消耗资源。

不那么消耗资源的一种方法是：

+ 没有定时器来操作 energy
+ 在增减 energy 数量后，保存 energy 数量和操作时间
+ 在查询和增减 energy 数量前，根据 energy 数量、增减速度、容量、记录的操作时间，计算出当前的实际 energy，代码类似于：

```ts
function getRealEnergy(energy: number, rate: number, limit: number, time: number) {
    if (energy >= limit) {
        return limit;
    }
    energy += Math.floor((Date.now() - time) * rate);
    return energy >= limit ? limit : energy;
}
```
