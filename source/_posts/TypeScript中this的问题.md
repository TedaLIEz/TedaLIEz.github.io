---
title: TypeScript中this的问题
date: 2019-05-25 15:52:23
tags: TypeScript
---


typescript中对于this的转译有点意思，例如下面一段代码

```typescript
class Foo {
    bar = 123;
    bas() {
        setTimeout(function() {
            console.log('this is', this);
            console.log(this.bar);
        }, 100);
    }
}

var foo = new Foo();
foo.bas();
```

这段代码中，this将会指向settimeout的this，即window,那么this.bar就会是undefined



转译结果

```javascript
var Foo = (function () {
    function Foo() {
        this.bar = 123;
    }
    Foo.prototype.bas = function() {
        setTimeout(function () {
            console.log('this is', this);
            console.log(this.bar);
        }, 100);
    }
    return Foo;
})();

var foo = new Foo();
foo.bas();
```



为了解决这个问题，最快的一个方法就是使用ts的lambda表达式

```typescript
class Foo {
    bar = 123;
    bas() {
        setTimeout(() => {
            console.log('this is', this);
            console.log(this.bar);
        }, 100);
    }
}

var foo = new Foo();
foo.bas();
```





转译结果

```javascript
var Foo = (function () {
    function foo() {
        this.bar = 123;
    }
    Foo.prototype.bas = function() {
        var _this = this; // tricky part
        setTimeout(function () {
            console.log('this is', _this);
            console.log(_this.bar);
        }, 100);
    };
    return Foo;
})();

var foo = new Foo();
foo.bas();
```



类似的还有



```typescript
class Foo {
    bar = 123;
    bas() {
        console.log('this is', this);
        console.log(this.bar);
    }
}

var foo = new Foo();
var bas = foo.bas;
bas();
```

这里我们把Foo的成员函数传给了bas变量，然后bas变量在外部调用。这个类似的场景还有将一个类的成员变量函数作为参数传给外部（回调等场景）

转译结果

```javascript
var Foo = (function () {
    function foo() {
    	this.bar = 123;
	}
	
	Foo.prototype.bas = function() {
        console.log('this is', this);
        console.log(this.bar);
	};
	return bas;
})();

var foo = new Foo();
var bas = foo.bas;
bas();
```

这个时候，bas调用的this将会指向全局变量，即window



解决方案同样是使用lambda表达式



```typescript
class Foo {
    bar = 123;
    
    bas = () => {
        console.log('this is', this);
        console.log(this.bar);
    }
}

var foo = new Foo();
var bas = foo.bas;
bas();
```



转译结果

```javascript
var Foo = (function () => {
    function Foo() {
        var _this = this;
        this.bar = 123;
        
        this.bas = function() {
            console.log('this is', _this);
            console.log(_this.bar);
        };
    }
    return foo;
})();

var foo = new Foo();

var bas = foo.bas;
bas();
```

