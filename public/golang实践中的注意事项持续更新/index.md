# Golang实践中的注意事项（持续更新）


# defer实践遇到的坑

defer用来声明一个延迟函数，把这个函数放入到一个栈上，当外部的包含方法return之前，返回参数到调用方法之前调用，也可以说是运行到最外层方法体时调用。我们经常用他来做一些资源的释放，比如关闭io操作。

但是在两个场景下需要特别注意：1. defer涉及参数传递 2. defer涉及对返回值的操作

## 参数传递

在定义defer时传递的参数是作为值拷贝传递的，也就是说如果原来的值发生变化，不会影响defer中的值。

```go
func main() {
	a, b, c := 1, 2, 3
	defer func(a, b, c int) {
		fmt.Println(a, b, c)
	}(a, b, c)
	a, b, c = 4, 5, 6
	defer fmt.Println(a, b, c)
	fmt.Println(a, b, c)
}
```

输出：

```cmd
4 5 6
4 5 6
1 2 3
```

当遇到在一个方法中对原数据进行改动，但是最后又想要使用原数据进行一些操作时，可以用到这个特性。

注意：如果defer传递的是指针，那么，外面的改动还是会影响defer内部的值的，道理很简单，指向同一块空间，自然是同步的。

## 对返回值操作

在defer中对返回值进行操作，大致根据返回值类型可以分为两大类：返回匿名变量和返回命名变量。

对于返回匿名变量，无论defer中怎么操作，都不会改变最终的返回值。

对于命名变量，只有一种情况不会修改返回值，那就是通过只拷贝将变量传递到defer函数内。

```go
func main() {
	fmt.Println(deferFun())
	fmt.Println(deferFun1())
	fmt.Println(deferFun2())
	fmt.Println(deferFun3())
	fmt.Println(deferFun4())
	fmt.Println(deferFun5())
}
func deferFun() (a, b, c int) {
	a, b, c = 1, 2, 3
	defer func() {
		a, b, c = 4, 5, 6
	}()
	return
}

func deferFun1() (a, b, c int) {
	a, b, c = 1, 2, 3
	defer func(a, b, c int) {
		a, b, c = 4, 5, 6
	}(a, b, c)
	return
}
func deferFun2() (a, b, c int) {
	a, b, c = 1, 2, 3
	defer func(a, b, c *int) {
		*a, *b, *c = 4, 5, 6
	}(&a, &b, &c)
	return
}
func deferFun3() (int, int, int) {
	a, b, c := 1, 2, 3
	defer func() {
		a, b, c = 4, 5, 6
	}()
	return a, b, c
}
func deferFun4() (int, int, int) {
	a, b, c := 1, 2, 3
	defer func(a, b, c int) {
		a, b, c = 4, 5, 6
	}(a, b, c)
	return a, b, c
}
func deferFun5() (int, int, int) {
	a, b, c := 1, 2, 3
	defer func(a, b, c *int) {
		*a, *b, *c = 4, 5, 6
	}(&a, &b, &c)
	return a, b, c
}
```

输出：

```cmd
4 5 6
1 2 3
4 5 6
1 2 3
1 2 3
1 2 3
```

> 原理

对返回值的操作，归根结底是defer和return的顺序问题，实际上defer函数的执行既不是在return之后也不是在return之前，而是return语句包含了对defer函数的调用，即return会被翻译成下面几条伪指令：

1. 保存返回值到栈上，如果是匿名变量，需要在栈上定义变量并赋值（程序员无需理会）
2. 如果有defer函数，则调用defer函数
3. 调整函数栈
4. 返回（如果是匿名变量，直接返回新定义的变量，如果是命名变量，直接返回命名变量）

命名变量返回时，不会创建新的变量，所以defer的修改会返回去。
而匿名变量，会创建新的变量，defer中的修改，还是修改原来的变量，所以修改不能返回去。




