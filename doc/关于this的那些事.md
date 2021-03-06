####1.2误解
#####1.2.1 指向自身
人们很容易把this理解成指向函数自身，这个推断从英语的语法角度来说是说得通的。
那么为什么需要从函数内部引用函数自身呢？常见的原因是递归（从函数内部调用这个函数）或者可以写一个在第一次调用后自己解除绑定的事件处理器。
javascript的新手认为，既然函数看作一个对象，javascript中得所有函数都是对象，那就可以在调用函数时存储状态(属性的值)。这是可行的，有些时候也确实有用，但是在本书即将介绍的许多模式中你会发现，除了函数对象，还有许多更合适存储状态的地方。
不过现在我们先来分析一下这个模式，让大家看到`this`并不像我们所想的那样指向函数本身。
我们想要记录一下函数`foo`被调用的次数，思考一下下面的代码：
<pre>function foo(num){
	console.log("foo: " + num);
	this.count ++;
}

foo.count = 0;
var i;
for (i = 0; i &lt; 10; i++){
	if(i > 5 ){
		foo（i）
	}
}
console.log(foo.count);
</pre>
consol.log语句产生了4条输入，证明foo确实被调用了4次，但是foo.count仍然是0。显然从字面意思来理解this的错误的。
执行foo.count = 0 时，的确向函数对象foo添加一个属性count。但是函数内部代码this.count中得this并不是指向那个函数对象，所以虽然属性名相同，根对象却不相同，困惑随即产生。
> 负责开发者一定会问，如果我增加的count属性和预期的不一样，那我增加哪个count？实际上，如果他深入探索的话，就会发现这段代码在无意中创建了一个全局变量count，它的值为NaN。当然，如果他发现了这个奇怪的结果，那一定会问，为什么它是全局，为什么它的值是NaN而不是其他更合适的值？

遇到这样的问题，许多开发者并不会深入思考为什么this的行为和预期的不一致，也不会试图回答哪些很难解决但非常重要的问题。他们只会回避这个问题并使用其他方法来达到此目的，比如创建另一个带有count属性的对象。
<pre>
function foo(num) {
  console.log("foo: "+ num);
  data.count++;
}
var data = {
count:0
}
var i;
for(i=0; i&lt;10; i++){
	if(i > 5) {
		foo(i)
	}
}

console.log(data.count);
</pre>
从某种角度来说这个方法确实解决了问题，但可惜他忽略了真正的问题，无法理解this的含义和工作原理，而是返回了舒适区，使用了一种更熟悉的技术，词法作用域。
如果要从函数对象内部引用它自身，那只使用this是不够的。一般来说你需要通过一个指向函数对象的词法标示符来引用它。
思考一下下面这个函数。
<pre>
function foo(){
	foo.count = 4;
}
setTimeout(function(){
	//匿名，函数无法指向自身
}, 10);
</pre>
第一个函数称为具名函数，在它内部可以使用foo来引用自身。
但是在第二个例子中，传入setTimeout的回调函数没有标识符，这种函数称之为匿名函数，因此无法从函数内部引用自身。
> 还有一种传统的但是现在已经被弃用和批判的用法，是使用arguments.callee来引用当前正在运行的函数对象。这是唯一一种可以从匿名函数对象内部引用自身的方法。然而，更好地方式是避免使用匿名函数，至少在需要自引用时使用具名函数，arguments.callee已经被弃用，不应该在使用它。

所以，对我我们的例子来说，另一种解决方法是使用foo标识符替代this来引用函数对象：
<pre>
function foo(num){
	console.log("foo"+ num);
	foo.count ++;
}
foo.count = 0;
var i;
for(i = 0; i &lt; 10; i++){
	if(i > 5){
	foo (i);
	}
}
console.log(foo.count);
</pre>
 然而，这种方法同样回避了this的问题，并且完全依赖变量foo的词法作用域。另一种是强制this指向foo函数对象：
 `function foo (num) {
		console.log("foo" + num);
		//记录foo被调用的次数
		//注意，在当前的调用方式下，this确实指向foo
		this.count ++;
}
foo.count = 0;
var i;
for (i = 0; i < 10; i++){
if(i > 5){
foo.call(foo, i);
}
}
console.log(foo.count);`

这次我们接受了this，没有回避它。如果你仍然感到困惑的话，不用担心，之后我们会详细解释具体的原理。
#####1.2.2 它的作用域
第二种常见的误解是，this指向函数的作用域。这个问题有点复杂，因为在某种情况下它是正确的，但是在其他情况下他却是错误的。
需要明确地市，this在任何情况下都不指向函数的词法作用域。在javascript内部，作用域确实和对象类似，可见的标识符都是它的属性。但是作用域对象无法通过javascript代码访问，它存在于javascript引擎内部。
思考一下下面的代码，它视图跨越边界，使用this来隐式引用函数的词法作用域
<pre>
function foo () {
	var a = 2;
	this.bar();
}
function bar () {
	console.log(this.a);
}

foo();
</pre>
这段代码中得错误不止一个。虽然这段代码看起来好像是我们故意写出来的例子，但是实际上它是出自一个公共社区中互助论坛的精华代码。这段代码非常完美的展示了this多么容易误导人。
首先，这段代码视图通过this.bar()来引用bar()函数。这是绝对不可能成功的，我们之后会解释原因。调用bar()最自然的方法是省略前面的this，直接使用词法引用标识符。
此外，编写这段代码的开发者还试图使用this联通foo()和bar()的词法作用域，从而让bar()可以访问foo()作用域的变量a。这是不可能实现的，你不能使用this来引用一个词法作用域内部的东西。
每当你想要把this和词法作用域的查找混合使用时，一定要提醒自己，这是无法实现的。
####1.3 this 到底是什么
排除了一些误解之后，我们来看看this到底是一种什么样的机制。
之前我们说过this是在运行时绑定的，并不是在编写时绑定的，它的上下文取决于函数调用是的各种条件。this的绑定和函数声明的位置没有任何关系，只取决于函数的调用方式。
当一个函数被调用时，会创建一个活动记录，有时也称为执行上下文。这个记录会包含函数在哪里被调用，函数调用方法，传入的参数等信息。this就是记录的其中一个属性，会在函数执行的过程中用到。
####1.4 小结
对于那些没有投入时间学习javascript的开发者来说，this的绑定一直是一件非常令人困惑的事。this是非常重要的，但是猜测，尝试并出错和盲目的从stackOverflow上复制粘贴答案并不能让你真正理解this的机制。
学习this的第一步是明白this既不指向函数自身，也不指向函数的词法作用域，你也许被这样的解释误导过，但其实它们都是错误的。
this实际上是在函数调用时发生的绑定，它指向什么完全取决于函数在哪里被调用。
