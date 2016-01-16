---
layout: post
title:  JavaScript Basics
date:   2016-1-17 00:00
categories: coding
---

JavaScript借鉴Java的语法最多，但也受awk,perl和python影响。它是大小写敏感的，使用Unicode字符集。在JavaScript中，语句被称为statements，并用分号分隔(;)。
JavaScript的脚本的源文本从左到右扫描，并转换成由令牌，控制字符，行结束符，注释或空白组成的输入元素序列。ECMAScript中还定义了某些关键字和字面值，并具有分号自动插入(ASI)来结束语句。但是，建议在语句结束时添加分号以避免副作用。


##1.注释（Comments）

注释语法跟C++和许多其他语言相同：

	// 单行注释

	/* 这是多行注释
	   多行注释
	*/

	/* /* 嵌套注释 */ 语法错误 */

##2.声明（Declarations）

JavaScript有三种声明方式：

var: 声明变量，可选初始化值。

let: 声明块范围局部变量，可选初始化值。

const: 声明一个只读命名常量。

变量在应用程序中，你使用变量来为值命名。变量名称或称为标识符，需要遵守一定的规则。在JavaScript语言中，一个标识符必须以字母、下划线（_）或者美元（$）符号开头后续的字符可以包含数字（0-9）。

声明变量可以使用下面两种方式：

* 使用关键字var。例如，var x = 42。这个语法可以同时用来声明局部和全局变量。

* 直接赋值。例如，x = 42。这样就声明了一个全局变量并会导致JavaScript编译时产生一个严格警告。因而应避免使用这种非常规格式。

用var或let声明时未赋初值的变量，值会被设定为undefined

	var a;
	console.log("The value of a is " + a); // logs "The value of a is undefined"
	console.log("The value of b is " + b); // throws ReferenceError exception

可以使用undefined来确定变量是否已赋值。以下代码中，变量input未被赋值，因而if条件语句的求值结果是true.

	var input;
	if (input === undefined) {
		doThis();
	} else {
		doThat();
	}

undefined值在布尔类型环境中会被当作false。例如，下面的代码将运行函数myFunction，因为数组myArray中的元素未被赋值：

	var myArray = new Array();
	if (!myArray[0]) myFunction();

数值类型环境中undefined值会被转换为NaN（Not a Number）。当你对一个空变量求值时，空值null在数值环境中会被当作0来对待，而布尔类型环境中会被当作false

	var a;
	a + 2 = NaN;
	var n = null;
	console.log(n*32); // log 0

在所有函数之外声明的变量，叫做全局变量，因为它可被当前文档中的其他代码访问。在函数内部声明的变量，叫做局部变量，因为它只能在该函数内部访问。

JavaScript变量的另一特别之处在于，你可以引用稍后声明的变量，而不会引发异常。这一概念称为变量声明提升(hoisting)。JavaScript变量感觉上是被“举起”或提升到了所有函数和语句之前。但是，提升后的变量将返回undefined值，即使在使用或引用后面存在声明和初始化操作，仍将返回undefined值。由于存在变量声明提升，一个函数中所有的var语句应该尽可能放在接近函数顶部的地方。这大大提升了程序的清晰度。

	/**
	 * Example 1
	 */
	console.log(x === undefined); // log "true"
	var x = 3;

	/**
	 * Example 2
	 */
	// will return a value of undefined
	var myvar = "my value";
	(function() {
		console.log(myvar); // undefined
		var myvar = "local value";
	})();


上面的例子也可以写作：

	/**
	 * Example 1
	 */
	var x;
	console.log(x === undefined); // logs "true"
	x = 3;
	/**
	 * Example 2
	 */
	var myvar = "my value";
	(function() {
		var myvar;
		console.log(myvar); // undefined
		myvar = "local value";
	})();

全局变量实际上是全局对象的属性。在网页中，全局对象是 window，所以你可以用形如window.variable的语法来设置和访问全局变量。

关键字const创建一个只读常量。常量标识符的命名规则和变量的相同：必须以字母、下划线或美元符号开头并可以包含有字母、数字或下划线。常量不可以通过赋值改变其值，也不可以在脚本运行时重新声明。常量，包括全局常量，都必须带const关键字，除此之外，常量的作用域规则与 let 块级作用域变量相同。若const关键字被省略了，该标识符将被视为变量。在同一作用域中，不能用与变量或函数同样的名字来命名常量。

##3.数据结构和类型

JavaScript语言拥有下面七种不同的类型的值：

* 六种是原型的数据类型:

Boolean 布尔值，true和false。

null 一个表明null值的特殊关键字。

undefined 变量未定义时的属性。

Number 表示数字，例如：42或者3.14159。

String 表示字符串，例如："hello world"。

Symbol 一种数据类型，它的实例是唯一且不可改变的。

* Object对象

##4.字面值（literals）

* 数组字面值

数组字面值是一个封闭在方括号对([])中的包含有零个或多个表达式的列表，其中每个表达式代表数组的一个元素。当你使用数组字面值创建一个数组时，该数组将会以指定的值作为它的元素进行初始化，而其长度被设定为元素的个数。如：

	var coffees = ["French Roast", "Colombian", "Kona"];

若在顶层（全局）脚本里用字面值创建数组，JavaScript语言会在每次对包含该数组字面值的表达式求值时解释该数组。另一方面，在函数中使用的数组，将在每次调用函数时被创建一次。

* 布尔类型有两种字面值：true和false。

*　整数

整数可以被表示成十进制（基数为10）、十六进制（基数为16）以及八进制（基数为8）。

十进制整数字组成的数字序列，不带前导0（零）。

带前导0（零）的整数字面值表明它是八进制。八进制整数只能包括数字0-7。

前缀0x或0X表示十六进制。十六进制整数，可以包含数字（0-9）和字母a~f或A~F。

	0, 117 and -345 (decimal, base 10)
	015, 0001 and -077 (octal, base 8) 
	0x1123, 0x00111 and -0xF1A7 (hexadecimal, "hex" or base 16)

* 浮点数字面值

浮点数字面值可以有以下的组成部分：

一个十进制整数，它可以带符号（即前面的“+”或“ - ”号），

一个小数点（“.”），

一个小数部分（由一串十进制数表示），

一个指数部分。

指数部分是以“e”或“E”开头后面跟着一个整数，可以有正负号（即前面写“+”或“-”）。一个浮点数字面值必须至少有一位数字，后接小数点或者“e”（大写“E”也可）组成。一些浮点数字面值的例子，如3.1415，-3.1E13，.1e12以及2E-12。

	[digits][.digits][(E|e)[(+|-)]digits]

* 对象字面值

对象字面值是封闭在花括号对({})中的一个对象的零个或多个"属性名-值"对的（元素）列表。你不能在一条语句的开头就使用对象字面值，这将导致错误或非你所预想的行为，因为此时左花括号（{）会被认为是一个语句块的起始符号。

	var Sales = "Toyota";

	function CarTypes(name) {
  		return (name == "Honda") ?
    	name :
    	"Sorry, we don't sell " + name + "." ;
	}

	var car = { myCar: "Saturn", getCar: CarTypes("Honda"), special: Sales };

	console.log(car.myCar);   // Saturn
	console.log(car.getCar);  // Honda
	console.log(car.special); // Toyota

* 字符串字面值

字符串字面值可以包含有零个或多个字符，由双引号（"）对或单引号（‘）对包围。字符串被限定在同种引号之间；也即，必须是成对单引号或成对双引号。如："foo"；"one line \n another line"等。

你可以在字符串字面值上使用字符串对象的所有方法——JavaScript会自动将字符串字面值转换为一个临时字符串对象，调用该方法，然后废弃掉那个临时的字符串变量。你也能用对字符串字面值使用类似String.length的属性：

	"John's cat".length

JavaScript 特殊字符：
	
| character | meaning         																						|
| --------- | ------------------------------------------------------------------------------------------------------|
| \0 	 	| 空字节																								|
| \b 		| Backspace 																							|
| \f 	 	| Form feed	 																							|
| \n 	    | New line 																								|
| \r 		| Carriage return　																						|
| \t 	 	| Tab 		 																							|
| \v 	    | Vertical tab																							|
| \' 		| Apostrophe or single quote																			|
| \" 	    | Double quote																							|
| \\ 		| Backslash character (\).　																			|
| \XXX 	 	| The character with the Latin-1 encoding specified by up to three octal digits XXX between 0 and 377.	|
| \xXX 	    | VThe character with the Latin-1 encoding specified by the two hexadecimal digits XX between 00 and FF.|
| \uXXXX	| The Unicode character specified by the four hexadecimal digits XXXX. 									|

##5.Unicode编码

Unicode是一种通用字符编码标准，用于世界上主要书面语言的交换和显示。它涵盖美洲，欧洲，中东，非洲，印度，亚洲和环太平洋地区的语言，还包括古文字和技术符号。 Unicode允许多语言文本的交换、处理和显示，以及通用的技术和数学符号的使用。它有望解决多语言计算的国际化问题，如不同国家的文字标准，以解决国际化问题。但目前，并非所有的现代或古老的文字都得到了支持。Unicode字符集可用于所有已知的编码。 Unicode字符集是在ASCII（美国信息交换标准码）字符集后建立的，它为每个字符设定一个数值和名称。字符编码指定字符的本体(identity)和数值（编码位置），以及这个数值的位(bit)表示。 16位的数字值（编码值）定义为一个带前缀U的十六进制数，例如，U+0041代表A。这个值的唯一名称是大写的拉丁字母A。你可以在字符串字面值、正则表达式和标识符中使用Unicode转义序列。转义序列由6个ASCII字符构成：\u和一个4位十六进制数字。例如，\u00A9表示版权符号。每一个Unicode转义序列在JavaScript中被解释为一个字符。

JavaScript中Unicode转义序列的用法和Java不同。在JavaScript中，转义序列绝不会首先被理解为一个特殊字符。例如，在一个字符串的换行符转义序列不终止字符串，它被解释的功能。Javascript忽略注释中的任何转义序列。在Java中，如果在单行注释中使用转义序列，则它被解释为一个Unicode字符。对于一个字符串字面量，Java编译器首先解释转义序列。例如，如果在JavaScript中使用一个行结束符转义字符（例如，\u000A），它终止字符串文字。而在Java中这样做将导致一个错误，因为(Unicode)换行符不允许出现在字符串字面值中，必须使用\n换行。在JavaScript中，转义序列的效果和\n一样。