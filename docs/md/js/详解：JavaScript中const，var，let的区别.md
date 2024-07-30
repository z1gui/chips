> # 详解：javascript中 const，var，let的区别

var、const、let 同样都是声明变量的关键词。

# 一、var 和 let 区别

## 作用域

var 的作用域只能是全局或者是整个函数块，而 let 的作用域既可以是全局变量或者是整个函数，还可以是 if, while, switch 限定的代码块。

```javascript
function varTest() {  
	var a = 1;  
	{    
		var a = 2; // 函数块中，同一个变量 
		console.log(a); // 2  
	}  
	 console.log(a); // 2
}

 function letTest() {  
	 let a = 1;  
	 {    
		 let a = 2; // 代码块中，新的变量    
		 console.log(a); // 2  
	 }  
	 console.log(a); // 1
 }
 varTest();
 letTest();
```

let 声明的变量，可以比 var 声明的变量的作用有更小的限定范围，更加灵活。

## 重复声明

在同一个作用域中，var 允许重复声明，但是 let 不允许重复声明。

```javascript
var a = 1;
var a = 2;
console.log(a) // 2
function test() { 
 var a = 3;  
 var a = 4;  
 console.log(a) // 4
 }
 test()


if(false) {  
	let a = 1;  
	let a = 2; // SyntaxError: Identifier 'a' has already been declared}
}

switch(index) {  
	case 0:    
		let a = 1;  
		break;  
	default:    
		let a = 2; // SyntaxError: Identifier 'a' has already been declared    
		break;
}
```

## 绑定全局变量

var 在全局环境声明变量，会在全局对象里新建一个属性，而 let 在全局环境声明变量，则不会在全局对象里新建一个属性。

```javascript
var foo = 'global'
let bar = 'global'
console.log(this.foo) // global
console.log(this.bar) // undefined
```


![[d81d47f1a51cc2ecfe16e8966e7949b8_MD5.jpeg]](img/d81d47f1a51cc2ecfe16e8966e7949b8_MD5.jpeg)

由上图可知，let 在全局环境声明变量 bar 保存在\[\[Scopes]]\[0]: Script 这个变量对象的属性中，而 \[\[Scopes]]\[1]: Global 就是我们常说的全局对象。

## 变量提升和暂存死区

了解变量提升，就需要了解到上线文和变量对象。详见[详解：javascript 创建执行释放过程](/md/js/详解：JavaScript创建执行释放过程.md)

### 变量提升

所有使用 var 声明的变量都会在执行上下文的创建阶段时作为变量对象的属性被创建并初始化，这样才能保证在执行阶段能通过标识符在变量对象里找到对应变量进行赋值操作等。即 var 在声明变量构建变量的时：
1. 由名称和 undefined（形参）组成一个变量对象的属性创建（创建并初始化）
2. 如果变量名称和之前的形参或者函数相同，则变量声明不会干扰已经存在的这类属性。

```javascript
console.log(a) // undefined
var a = 1;
console.log(a) // 1
```

为什么 var 变量可以在声明之前使用，因为使用是在执行阶段，而在此之前的创建阶段就已经将声明的变量添加到了变量对象中，所以执行阶段通过标识符可以在变量对象中查找到，也就不会报错。

### 暂存死区

其实 let 也存在与 var 类似的“变量提升”过程，但与 var 不同的是其在执行上下文的创建阶段，只会创建变量而不会被初始化（undefined），并且 ES6 规定了其初始化过程是在执行上下文的执行阶段（即直到它们的定义被执行时才初始化），使用未被初始化的变量将会报错。

>let and const declarations define variables that are scoped to the running execution context’s LexicalEnvironment. The variables are created when their containing Lexical Environment is instantiated but may not be accessed in any way until the variable’s LexicalBinding is evaluated. A variable defined by a LexicalBinding with an Initializer is assigned the value of its Initializer’s AssignmentExpression when the LexicalBinding is evaluated, not when the variable is created. If a LexicalBinding in a let declaration does not have an Initializer the variable is assigned the value **undefined** when the LexicalBinding is evaluated.

在变量初始化前访问该变量会导致 ReferenceError，因此从进入作用域创建变量，到变量开始可被访问的一段时间（过程），就称为暂存死区(Temporal Dead Zone)。

```javascript
console.log(bar); // undefined
console.log(foo); // ReferenceError: foo is not defined
var bar = 1;
let foo = 2;


var foo = 33;
	{  
		let foo = (foo + 55); // ReferenceError: foo is not defined
	}

```
## 小结

1. var 声明的变量在执行上下文创建阶段就会被「创建」和「初始化」，因此对于执行阶段来说，可以在声明之前使用。
2. let 声明的变量在执行上下文创建阶段只会被「创建」而不会被「初始化」，因此对于执行阶段来说，如果在其定义执行前使用，相当于使用了未被初始化的变量，会报错。

## 二、let 和 const 区别

const 与 let 很类似，都具有上面提到的 let 的特性，唯一区别就在于 const 声明的是一个只读变量，声明之后不允许改变其值。因此，const 一旦声明必须初始化，否则会报错。

示例代码：

```javascript
let a;
const b = "constant"
a = "variable"
b = 'change' // TypeError: Assignment to constant variable
```
**如何理解声明之后不允许改变其值？**

其实 const 其实保证的不是变量的值不变，而是保证变量指向的内存地址所保存的数据不允许改动（即栈内存在的值和地址）。

javascript 的数据类型分为两类：原始值类型和对象（Object类型）。

对于原始值类型（undefined、null、true/false、number、string），值就保存在变量指向的那个内存地址（在栈中），因此 const 声明的原始值类型变量等同于常量。

对于对象类型（object，array，function等），变量指向的内存地址其实是保存了一个指向实际数据的指针，所以 const 只能保证指针是不可修改的，至于指针指向的数据结构是无法保证其不能被修改的（在堆中）。

示例代码：

```javascript
const obj = {  value: 1}
obj.value = 2;
console.log(obj) // { value: 2 }obj = {}
```

参考资料：

> [深入理解 JS：var、let、const 的异同]( https://zhuanlan.zhihu.com/p/556482226?utm_id=0 )

