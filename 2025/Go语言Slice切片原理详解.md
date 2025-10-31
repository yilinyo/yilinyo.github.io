# Go 语言的 slice 详解

## go语言切片底层结构

slice 是 go 语言的可变数组，维护了 一个 结构体 如下

```go
type slice struct {
   array unsafe.Pointer
   len   int
   cap   int
}
```

slice 结构体 由一个指向 数组的指针和长度、容量构成 指针指向一个底层数组，len标识切片长度，标识切片的实际长度，cap标识底层数组的最大长度，若len\<cap，则数组中属于切片的只有 len长度，如下图示意图，slice1只有\[0,1] 两个元素,后面的2，3，4非切片的元素

![.png](https://s2.loli.net/2025/10/31/XJlK4W3dNo95zPM.png)

## 切片的复制与截取

将切片进行复制（赋值一个新变量），其实就是创建一个新的slice结构体，其中array指针指向被复制的结构体指向的底层数组

![.png](https://s2.loli.net/2025/10/31/imWnsAthq5NO8wD.png)

显然，当底层数组指向相同那么修改slice2的某个元素，同样也会影响到slice1，如下列代码所示

```go
func main() {
    arr1 := make([]int, 0, 4)
	arr1 = append(arr1, 0, 1)
	arr2 := arr1
	arr2[0] = -1
	fmt.Println(arr1, arr2)
	}
    // [-1,1] [-1,1]
```

```go
func main() {
	arr1 := make([]int, 0, 5)
	arr1 = append(arr1, 0, 1, 2, 3, 4)
	arr2 := arr1[1:3]
	fmt.Printf("arr1=%v, addr1=%p, len1=%d, cap1 = %d\n", arr1, arr1, len(arr1), cap(arr1))
	fmt.Printf("arr2=%v, addr2=%p, len1=%d, cap2 = %d\n", arr2, arr2, len(arr2), cap(arr2))
	arr2[0] = -1
	fmt.Printf("arr1=%v, addr1=%p\n", arr1, arr1)
	fmt.Printf("arr2=%v, addr2=%p\n", arr2, arr2)
}
	//arr1=[0 1 2 3 4], addr1=0x1400001e180, len1=5, cap1 = 5
	//arr2=[1 2], addr2=0x1400001e188, len1=2, cap2 = 4     容量是原容量减去被截取的元素数量
	//arr1=[0 -1 2 3 4], addr1=0x1400001e180
	//arr2=[-1 2], addr2=0x1400001e188
```

同样的当进行切片截取，底层的数组是不会变的，新的slice结构体的array指针会指向被截取的数组的开始位置的地址

## 追加元素到切片当中

向切片中追加元素，使用append(slice,i),Go 会检查底层数组是否有足够的容量来容纳新的元素。如果有足够的容量，新元素会被添加到底层数组的末尾，切片的长度会增加。如果没有足够的容量，就需要进行扩容。

如果容量满足：

则不会进行扩容，那么append操作会直接修改slice 的len，以及在**指向底层数组后直接追加元素**

如果容量不满足：

则会进行扩容，首先会在**底层创建一个新的数组（数组大小由此次追加元素和现有容量决定），将原底层数组的值全部拷贝到新的数组里**，然后再修改slice结构体里的array指向新的数组以及len以及cap容量。

![append.png](https://s2.loli.net/2025/11/01/EQBuadUwzWcjGtO.png)

```go
func main() {
	arr1 := make([]int, 0, 2)
	arr1 = append(arr1, 0)
	//打印arr1 底层数组的地址，如果填&arr1 打印的是 结构体的地址
	fmt.Printf("arr1=%v, addr1=%p\n", arr1, arr1)
	arr1 = append(arr1, 1)
	fmt.Printf("arr1=%v, addr1=%p\n", arr1, arr1)
	arr1 = append(arr1, 2)
	fmt.Printf("arr1=%v, addr1=%p\n", arr1, arr1)
}
//arr1=[0], addr1=0x140000100a0
//arr1=[0 1], addr1=0x140000100a0
//arr1=[0 1 2], addr1=0x14000016040
```

在go1.18版本后扩容机制大概如下，[可以参考 Go 语言切片如何扩容？（全面解析原理和过程）](https://developer.aliyun.com/article/1509760)

![.png](https://ucc.alicdn.com/pic/developer-ecology/wrp43id6ygvkg_dbbb7a08cd394e32b906dbccc8e81997.png?x-oss-process=image%2Fresize%2Cw_1400%2Cm_lfit%2Fformat%2Cwebp)

所以可见，切片的一次扩容会进行数组的一次全值复制，所以在初始化切片的时候尽可能制定第三个参数估计一个恰当的容量，提前在内存分配合适的空间能够减少扩容时带来的开销。

下面我可以看一个例子看我们是否真的理解了。以下代码会打印的是一样的吗？

```go
func main(){
   arr1 := make([]int, 0, 4)
   arr1 = append(arr1, 1)
   arr2 := append(arr1, 2)
   arr3 := append(arr1, 3)
   fmt.Printf("arr1=%v, addr1=%p\n", arr1, &arr1)
   fmt.Printf("arr2=%v, addr2=%p\n", arr2, &arr2)
   fmt.Printf("arr3=%v, addr3=%p\n", arr3, &arr3)
   arr0 := make([]int, 0, 1)
   arr0 = append(arr0, 1)
   arr4 := append(arr0, 2)
   arr5 := append(arr0, 3)
   fmt.Printf("arr0=%v, addr0=%p\n", arr0, &arr0)
   fmt.Printf("arr4=%v, addr4=%p\n", arr4, &arr4)
   fmt.Printf("arr5=%v, addr5=%p\n", arr5, &arr5)
}

/**
arr1=[1], addr1=0x140000ba000
arr2=[1 3], addr2=0x140000ba018
arr3=[1 3], addr3=0x140000ba030
arr0=[1], addr0=0x140000ba090
arr4=[1 2], addr4=0x140000ba0a8
arr5=[1 3], addr5=0x140000ba0c0
**/
```
arr2 和 arr3 为什么都是[1,3];因为在进行对arr1追加元素2其实只是在原来的底层数组里面增加元素，因为初始的容量是4，append一个元素满足最大容量，所以实际上arr1、arr2、arr3结构体的数组指针指向的地址是同一个，但由于arr1的len是1，所以后续相对arr1进行append，其实都是加载底层数组array[1]上，则最后的arr3 会覆盖掉arr2 的append；但是第二种场景由于一开始的容量是1，进行append后容量是不够的需要扩容，扩容底层数组会构造新的所以arr0，arr4，arr5其实结构体指针指向的是不同数组，所以也不会发生覆盖

## 切片在底层数组基础上的完全复制
基于相同底层数组的复制有时候会不小心修改错数据，其实go 也提供了一个copy函数支持将切片复制是在内存空间直接复制一个一模一样的数组，来隔离切片复制后的数组共享。如果复制时长度溢出则会截断

```go

func main() {
   arr := []int{1, 2, 3, 4}
   arr1 := make([]int, 3)
   cnt := copy(arr1, arr)
   fmt.Printf("cnt=%d\n", cnt)
   fmt.Printf("arr1=%v\n", arr1)
}
//cnt=3
//arr1=[1 2 3]
```

