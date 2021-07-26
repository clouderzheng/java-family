### map 系列

#### hashmap  
1、new hashmap（） 不会创建数组，初始化会在第一次put的时候触发。  
2、此时若传入初始化容量，此时会计算大于等于该值的最小2的次方数 。  
```
这段代码通过不断的无符号位移，一步步的用最高位的值填充满所有位置，然后进行或运算，快速计算出最小2次方数
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```
3、上述计算的容量值暂时存入threshold，threshold（扩容阀值 = 容量 * loadFactor ）临时充当容量，在第一次put的时候替换为容量，并且计算出真正的阀值。  
4、扩容的同时，如果原数组不为空，还会重新计算hash，重新分配，这里因为扩容是2倍，所以这里有个很巧妙的地方，同一个数组节点下的链表，在重新hash之后的值要么继续在原index，要么在 index + oldcap（老数组容量）

```
这里是针对一个链表节点的展示，其他节点相同
Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
```

5、扩容条件
- 数组实际存储值个数大于阀值
- 链表长度大于8，准备转数组，但是数组长度低于64，此时认为是数组太小，造成hash冲突过多，会扩容，而不会转红黑树。
6、hashmap只能存一个key为null的值，因为key为null的，统一处理hashcode为0.
7、hashmap获取一个值，比较方法
```
这里其实只使用后半部分即可，个人觉得前半部分e.hash == hash是为了快速判断短路设计
if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
```

8、jdk8相对于jdk7改进：
- 底层数据结构由数组 + 链表 改成 数组 + 链表 + 红黑树的形式，主要解决hash冲突严重时，链表查询O(n) 优化为 O（logn）
- 优化计算初始化容量的方式，之前通过一个循环，不断左位移，得到值与初始化比较，直到找到一个大于初始化值的数，1.8之后通过高位不断右移，使高位1填充满低位，通过或运算快速得到一个最小2次方
- 优化了hashcode的计算方式，现在直接通过高位右移16位参与异或运算，快速得到hash值
- 扩容时的插入方式从头插法变成尾插法，避免了并发下面的死循环
- 扩容时的重新计算新index（数组位置）的方式从 hash & （cap【数组长度】 - 1） 变成 判断hash & oldcap的值，通过是否为0 的情况确认是原位置还是 原位置 + oldcap
