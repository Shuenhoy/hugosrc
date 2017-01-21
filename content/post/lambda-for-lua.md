+++
categories = ["Lua"]
tags = ["Lua"]
description = ""
slug = "lambda-for-lua"
draft = false
title = "Lua的Short Anonymous Function"
date = 2013-09-28T04:00:00Z
author = "Shuenhoy"

+++


Lua的function这个关键字有时候挺烦人的。比如我就想要个`f(a, b)=a+b`的函数就要写成

```lang-lua
function f(a,b) return a+b end
```

于是在boost::lambda的启发下，我利用lua的重载运算符的特性做了一个"Short Anonymous Function" 我暂时把它狂妄地叫做lambda了，不过我也没有仔细研究过不知道能不能称得上lambda。
<!--more-->
现在可以做到这些： 

### 普通的计算
```lua
local f1 = _1 + _2
print(f1(1,3)) --> 4
```

### 调用其他函数
```lua
local f2 = _call(print,1)
f2() --> 1
```
### 读取table值
```lua
local f3 = _index({1,2},1)
print(f3) -->1
```
### if 和 递归
```lua
local fbi = _if(_lt(_1,3)):
            _then(1):
            _else(_resuc(_1-1)+_resuc(_1-2))
print(fbi(40)) -->102334155
```

## 实现 
因为我的水平比较低，所以实现的也比较烂了。

为了方便起见我直接把lambda定义为表达式了，表达式本身也是一个递归的定义：
```lua
Exp={Exp1,Exp2,Operator}
```

表达式的基本单位就是`_1` `_2`这样的参数，参数也是表达式，但是它只需要返回在上下文(Context)中自身的值。

这样比如`_1+_2 * _3`这个表达式实际上就拆成了`{_1, {_2, _3, *}, +}`

在调用一个lambda时 （即使用形如`f()`的调用）时，会创建一个上下文环境，里面保存着参数，表达式本身。上下文环境会一直传到参数那里。接着表达式使用对Exp1和Exp2进行Operator操作，并返回结果到上一层。递归的为每一层都这样计算，直到表达式不可以继续计算为止。

比如`_1+_2 * _3`（即`{_1, {_2, _3, *}, +}`）调用参数为(2,3,4)时：

  * 建立上下文
  * 首先对`_1` 和`{_2, _3, *}`进行`+`运算
  * * `{_2, _3, *}` 是一个可以继续计算的表达式，对`_2` `_3`进行`*运算
  * * 从上下文中获得`_2`的值为3 `_3`的值为4，返回3*4=12
  * 从上下文中获得`_1`的值为2，并得到`{_2, _3, *}`的值为12 返回2*12=24

至于递归也很容易就实现了，只需要在调用()的时候把自身保存在context即可

### 存在问题 ###
 1. Lua的`__lt` `__le`以及`__eq`的重载只会返回boolean，不管你在函数里返回了什么，只要不是nil或false，在外面只会得到true。以及无法重载and or not
 2. 嵌套的定义lambda问题。`a = _index(_call(table.sort,_lt(_1,_2)),2)+_1` 怎么知道哪部分是 新的lambda？
 3. 运算速度太慢

对于第一点，除了改lua的源码给他打补丁以外的方法都比较蛋疼。我最后是实现了`_lt` `_le` `_rt` `_re` `_eq` `_and` `_or` `_not` 这几个函数。。。

对于第二点，我发现其实lamdba之间本身是没有区别的，他们的区别只在于得到的参数不同，也就是Context不同，而且这种不同只在计算的时候才表现出来。所以要实现区别不同的函数，只需要设置一个标志，然后告诉表达式“这个不是你的一部分，不用计算了！”就可以解决。

我的方法是定义一个lambda函数,用第二个返回值做标志，这样即使在普通的时候调用它也不会出现问题
```lua
function lambda(exp)
  return exp, NOT_EXEC
end

local f1 = _call(table.sort, lambda(_lt(_1, _2)))
local f2 = lambda(_1 + _2)
```

第三个问题比较严重，前面提到的fbi，在25的时候普通的函数耗时0s 而lambda却用了整整11s！
```lua
local fbi = _if(_lt(_1,3)):
            _then(1):
            _else(_resuc(_1-1)+_resuc(_1-2))
function fbin(n)
	if n<3 then
		return 1
	end
	return fbin(n-1)+fbin(n-2)
end
local t

t=os.time()
print(fbin(25))
print(os.time()-t) --> 0

t=os.time()
print(fbi(25))
print(os.time()-t) --> 11
```
这个问题确实不好办，但是考虑到这些lambda大概都没有副作用，所以我是用了一个表来存了参数和结果的对应关系。。。

使用`lambda()`建立lambda时就会自动开启这个效果


最后仍上实现代码，前面有50行左右lambda依赖的我自己写的类似IO语言的对象系统

欢迎吐槽

```lua
---------------- 功能函数部分----------------------
function table.eachi(t, f)
	for i, v in ipairs(t) do
		if f(i, v,t[i+1]) == false then break end
	end
end
Object = {property={}}
function Object:clone ( _t )
	_t.property = _t.property or self.property or {}
	setmetatable(_t, {__index = self.__index, __newindex = self.__newindex, __pa=self, __call = self.__call,
						__add = self.__add, __sub = self.__sub, __div = self.__div, 
						__mul = self.__mul, __pow = self.__pow, __mod = self.__mod,
						__eq = self.__eq, __lt = self.__lt, __le = self.__le} )

	if _t.property ~= self.property then
		setmetatable(_t.property, {__index = self.property})
	end
	if self.init then self.init(_t) end
	return _t
end
function Object:__index( key )
	local mt=getmetatable(self)
	
	if self.property[key] and self.property[key].get then 
		return self.property[key].get() 
	end
	if type(mt.__pa)=="table" then
		return mt.__pa[key]
	end
	return nil
end

function Object:__newindex( key,value )
	local mt=getmetatable(self)
	if self.property[key] and self.property[key].set then return self.property[key].set(self,value) end
	rawset(self,key,value)
end
function Object:is_a( pa )
	local mt=getmetatable(self)
	if mt.__pa==pa then return true end
	return mt.__pa:is_a(pa)
end

setmetatable(Object,{__index=Object.__index,__newindex=Object.__newindex})

table.eachi({"add","sub","div", "mul", "pow", "mod", "call", "eq" , "le", "lt"}, function(k, v)
	Object.property["__"..v] = {set = function(self, value)
		getmetatable(self)["__"..v] = value
		rawset(self, "__"..v, value)
	end}	
end)
------------------ 功能函数部分结束-----------------------
------------------ Lambda 部分开始 ------------------------
Lambda = Object:clone{
	__call = function(...)
		
	end
}
local op={
	add = function(a, b) return a + b end;	sub = function(a, b) return a - b end;
	div = function(a, b) return a / b end;	mul = function(a, b) return a * b end;
	mod = function(a, b) return a % b end;	pow = function(a, b) return a ^ b end;
	eq  = function(a, b) return a == b end;	lt  = function(a, b) return a <  b end;
	le  = function(a, b) return a <= b end;
}
local function exec(obj, context)
	if type(obj) == "table" and obj:is_a(Exp) then
		return obj:exec(context)
	end
	return obj
end
local NOT_CALL = {}
local VAR = {}
Exp = Lambda:clone{
	
	And = function(first, second, context)
		return exec(first, context) and exec(second, context)
	end;
	Or = function(first, second, context)
		return exec(first, context) or exec(second, context)
	end;

	Not = function(first, second, context)
		return not exec(first, context)
	end;

	Index = function(first, second, context)
		return exec(first, context)[exec(second, context)]
	end;

	IndexSet = function(first, second, context)
		local result = exec(second, context)
		first[1]:exec(context)[first[2]:exec(context)] = result
		return result
	end;

	Call = function(first, second, context)
		local args = {} 
		table.eachi(second, function(i, arg, n)
			if arg ~= NOT_CALL then
				if n ~= NOT_CALL then
					table.insert(args, exec(arg, context))
				else
					table.insert(args, arg)
				end
			end
		end)
		return exec(first, context)(unpack(args))
	end;

	exec = function(self, context)
		return self.operator(self.first, self.second,context)
	end;
	__call = function(self, ...)
		local context = {args = {...}, base = self}
		if self.memorize then
			self.memory = self.memory or {}
			local match = true
			local m = self.memory
			local ret
			for i=1, #context.args do
				m[context.args[i]] = m[context.args[i]] or {}
				m = m[context.args[i]]
			end
			m[context.args[#context.args]] = m[context.args[#context.args]] or {}
			m[context.args[#context.args]][VAR] = m[context.args[#context.args]][VAR] or exec(self, context)
			return m[context.args[#context.args]][VAR]
		end
		return exec(self, context)
	end;
}
table.eachi({"Add","Sub","Div", "Mul", "Pow", "Mod",  "Eq" , "Le", "Lt"},function(k, v)
	Exp[v] = function(first, second, context)
		return op[v:lower()](exec(first, context),exec(second, context))
	end;
	Exp["__"..v:lower()] = function(self, other)		
		return Exp:clone{first = self, second = other, operator = Exp[v]}
	end;
end)

local IfExp = Exp:clone{
	exec = function(self, context)
		if exec(self.exp, context) then
			return exec(self.thenexp, context)
		else
			return exec(self.elseexp, context)
		end
	end;
	_then = function(self, exp)
		self.thenexp = exp
		return self
	end;
	_else = function(self, exp)
		self.elseexp = exp
		return self
	end;
	_elseif = function(self, exp)
		self.elseexp = _if(exp)
		return self.elseexp
	end
}
local resuc = Exp:clone{
	exec = function(self, context)
		return context.base
	end
}
Argument = Exp:clone{
	exec = function(self, context)
		return context.args[self.index]
	end;
	__call = function(self, val)
		return val
	end;
}
function _indexset(obj, key, value)
	return Exp:clone{first = {obj, key}, second = value, operator = Exp.IndexSet}
end
function _call(func, ...)
	return Exp:clone{first = func, second = {...}, operator = Exp.Call}
end

local function getf1(name)
	return function(a, b)
		return Exp:clone{first = a, second = b, operator = Exp[name]}	
	end
end
_and   = getf1("And")   _or    = getf1("Or")
_not   = getf1("Not")   _index = getf1("Index")
_lt    = getf1("Lt")    _le    = getf1("Le")
_rt    = getf1("Rt")    _re    = getf1("Re")
_eq    = getf1("Eq")
function _if(exp)
	return IfExp:clone{exp = exp}
end
function _resuc(...)
	return _call(resuc,...)
end
function lambda(exp, nmemorize)
	exp.memorize = not nmemorize
	return exp, NOT_CALL
end

function args(index)
	return Argument:clone{index = index}
end

_1 = args(1)
_2 = args(2)
_3 = args(3)
```