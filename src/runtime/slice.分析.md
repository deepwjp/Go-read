### slice的底层实现

> slice 的底层数据是数组，slice 是对数组的封装，它描述一个数组的片段。两者都可以通过下标来访问单个元素。

数组是定长的，长度定义好之后，不能再更改。数组因为其长度是类型的一部分，如 [3]int 和 [4]int 就是不同的类型。

> 切片则非常灵活，它可以动态地扩容。切片的类型和长度无关。

> 数组就是一片连续的内存， slice 实际上是一个结构体，包含三个字段：长度、容量、底层数组。

```go
type slice struct {
	array unsafe.Pointer // 元素指针
	len   int // 长度 
	cap   int // 容量
}
```

> Slice的array指针指向该slice在底层数组中的首元素。

slice 的数据结构如下：

![切片数据结构](slice.分析.assets\55270142-876c2000-52d6-11e9-99e5-2e921fc2d430-1604560285211.png)

### 函数参数中的slice

> 底层数组是可以被多个 slice 同时指向的，因此对一个 slice 的元素进行操作是有可能影响到其他 slice 的。

>
> slice 其实是一个结构体，包含了三个成员：len, cap, array。分别表示切片长度，容量，底层数据的地址。
>

> 当 slice 作为函数参数时，就是一个普通的结构体。其实很好理解：若直接传 slice，在调用者看来，实参 slice 并不会被函数中的操作改变；若传的是 slice 的指针，在调用者看来，是会被改变原 slice 的。
>

> 值的注意的是，不管传的是 slice 还是 slice 指针，如果改变了 slice 底层数组的数据，会反应到实参 slice 的底层数据。为什么能改变底层数组的数据？很好理解：底层数据在 slice 结构体里是一个指针，仅管 slice 结构体自身不会被改变，也就是说底层数据地址不会被改变。 但是通过指向底层数据的指针，可以改变切片的底层数据，没有问题。
>

> 通过 slice 的 array 字段就可以拿到数组的地址。在代码里，是直接通过类似 `s[i]=10` 这种操作改变 slice 底层数组元素值。



### Slice扩容规则：

> 1. 预估扩容后的容量
>
>    ![image-20201104175717549](slice.分析.assets\image-20201104175717549.png)
>
> 2. newCap需要多大内存
>
>    预估容量*元素类型大小
>
> 3. 将预估申请的内存匹配到合适的内存规格
>
>    Go语言自身实现的内存管理模块会提前向操作系统申请一批内存，分成常用的规格管理起来，在申请内存时，他会匹配到足够大切最接近的规则
>
> PS：规格产生的源码 在   src/runtime/msize.go

```go
// 如果slice的容量不够用，就是用growslice扩容
// cap 是新slice需要的最小容量
func growslice(et *_type, old slice, cap int) slice {
	// 检测是否有数据竞争
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}
	// 如果新要扩容的容量比原来的容量还要小，这代表要缩容了，那么可以直接报panic了。
	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}
	// 如果当前切片的大小为0，还调用了扩容方法，那么就新生成一个新的容量的切片返回。
	if et.size == 0 {
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

	// 扩容
	newcap := old.cap
	//扩容是旧的大小的双倍扩容
	doublecap := newcap + newcap
	//如果需要的新cap比两倍大小还大就直接扩到新cap大小
	if cap > doublecap {
		newcap = cap
	} else {
		// 旧的slice元素在1024内就直接双倍大小
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// 如果 >=1024 就一直1.25倍扩大到比需要的cap大
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// 溢出
			// 直接扩容到cap
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	// 计算新的切片的容量，长度。
	// 不同size大小不同的迁移方法
	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
        // 匹配相应规格大小的内存
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}

	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
	if et.ptrdata == 0 {
		p = mallocgc(capmem, nil, false)
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
		}
	}
	// 迁移旧数据
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}

```

