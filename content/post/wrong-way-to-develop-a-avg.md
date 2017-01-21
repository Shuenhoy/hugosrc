+++
author = "Shuenhoy"
description = ""
slug = "wrong-way-to-develop-a-avg"
draft = false
title = "正确的AVG开发姿势 （编写中）"
date = 2014-06-04T05:55:58Z

+++

### 错误的姿势
```js
function main(){
	message("赵苑秀",
    	"同学你好啊,有时间帮忙吗",
        function(){
        	select(["好啊","唉……",/*(更多选项)*/],
            	function(ret){
                	switch(ret){
                    case 0:
                    	message("赵苑秀",
                        	"太感谢了！",
	                        function(){
                            	//……
							});
                    	break;
                    case 1:
                      	message("赵苑秀",
                        	"不行吗……？",
	                        function(){
                            	//……
							});
	                    break;
	               case 2:
	               case 3:
	               case 4://更多处理
                    }
                }
            });
        });
}
```

### 更加错误的姿势
```lua
function main()
	message("赵苑秀",
    	"同学你好啊,有时间帮忙吗",
        function()
        	select({"好啊","唉……",--[[(更多选项)]]},
            	function(ret)
            	    if ret==0 then
            	        message("赵苑秀",
                        	"太感谢了！",
	                        function()
                            	--……
							end)
            	    elseif ret==1 then
            	        message("赵苑秀",
                        	"唉不行吗……？",
	                        function()
                            	--……
							end)
            	    elseif ret==2 then
            	    elseif ret==3 then
            	    elseif ret==4 then
            	    ……
            	    end
                	
                end
            end)
        end)
end
```

### 姿势2
```lua
function Yes()
	message"赵苑秀：十分感谢"
end

function No()
	message"赵苑秀：唉不行吗……？"
end

function main()
	message"赵苑秀：你好啊同学有时间帮忙吗？"
	option = select("好啊","唉……")
	if option == 1 then
 		return Yes
	elseif option == 2 then
		return No
	end
end
```

#### 姿势2的完整AVG的样子
```lua
function A()
return B
end
function B()
retuen C
end
function C()
return D
end
function D()
return ……
end

```

### 问题在哪？
在某种程度上游戏的剧情与引擎实际上是**异步**的，也就是说剧情按照自己的顺序进行而引擎有另一套不变的循环。剧情会因为某些原因阻塞（主要为等待输入） 但引擎则不能在此时被阻塞。一种可行的方案是将每个阻塞的操作都加入引擎的主循环代码 但显然会造成复杂的问题。

上述的两种方案在本质上都是callback的，一步一个函数，等处理完了再调用，显而易见 这会使游戏剧情编写变得异常**复杂**。

在有coroutine的语言中 则可以通过coroutine来协调操作，可以获得好的体验。但是一旦使用coroutine 绝无再使用callback的必要（当然tail call与此根本毫无关系）

更为常见的方式则是写一个简单的剧情解析器，不会存在上述问题。(参加Project Shadow RPG、Project AlaalA 例子之后补上) 
