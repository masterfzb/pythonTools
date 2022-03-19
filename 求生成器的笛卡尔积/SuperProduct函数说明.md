# super_product
### 1.函数作用说明
函数效果与itertools.product相同，其作用是输入一组生成器，然后输出一组笛卡尔积。
但是itertools.product的实现方法如下：
```python
def product(*args, repeat=1):
    # product('ABCD', 'xy') --> Ax Ay Bx By Cx Cy Dx Dy
    # product(range(2), repeat=3) --> 000 001 010 011 100 101 110 111
    pools = [tuple(pool) for pool in args] * repeat
    result = [[]]
    for pool in pools:
        result = [x+[y] for x in result for y in pool]
    for prod in result:
        yield tuple(prod)
```

此函数的实现方法是先将所有的笛卡尔积求出，然后 保存在result内，每次生成器next时，就yield 其中的一个值。

此方法的优点是最开始生成result时可能会卡一段时间，但是卡完以后每次测试都是从已获取的结果里返回一个值，不再有大量的资源消耗。

缺点是过于消耗系统内存，假如单个生成器生成的值总量极大，那么最后result会占用的内存数量级很恐怖。在部分情况（如fuzzing测试中），这样的result是不可能存在的。

### 2.函数改写的思路
1.输入的是生成器函数而非生成器
2.先生成一组（初始值），然后while True执行以下动作：
3.对这组值的最后一个值开始next，假如最后一个值用完了，则换掉最后一个值的生成器，然后对前一个值进行next。
4.假如前一个值也next完了，则前一个值也换一个生成器，对前前一个值进行next，以此类推。
5.第一个值next完后，说明笛卡尔积求完了。返回一个Stop的flag 给while True过程让生成过程停止。

源码如下。
```python
def rabbithole(a,lis,org,i,hole_deep =0):
    #兔子洞的深度
    
    a[i] = lis[i]()#更新用坏的生成器
    
    if i != len(a)-1:
        org[i] = next(a[i])
    value = next(a[i-1],'end')#上一阶进一。
    
    if value ==org[i-1]:
        value = next(a[i-1],'end')#假如上一阶是一个新的生成器，那上一阶再进一一次，抵消掉0.
        
    if value != 'end':#假如本次的进一成功
        org[i-1] = value#则本次的值更新到org
        return a,lis,org,'continue',hole_deep
    else:
        if hole_deep == len(a)-2:
            return a,lis,org,'stop',hole_deep
        return rabbithole(a,lis,org,i-1,hole_deep +1)

def super_product(lis):
    org = []

    for i in range(len(lis)):
        a = lis[i]()
        org.append(next(a))
    a =[]
    for i in range(len(lis)):
        a.append(lis[i]())
    link = 'continue'
    hole_deep = 0
    i = len(a)-1
    
    while True:
        
        if link == 'stop':
            break
        
        hole_deep = 0
        value=next(a[i],'end')
        
        if value != 'end':
            org[i] =value
            yield org,link
        else:
            a,lis,org,link,hole_deep = rabbithole(a,lis,org,i)
```