JS原声Date类型方法的一些冷知识
===========================

由于`Date()`是JS原生函数，不同浏览器的解析器对其实现方式并不同，所以返回值也会有所区别。本文测试未特别申明浏览器的情况下，均是指`win7 x64` + `chrome 44.0.2403.155(正式版) m (32位)`版本。

## Date与new Date的区别

`Date()`直接返回当前时间字符串，**不管参数是number还是string**

```js

Date();
Date('sssss');
Date(1000);
//Fri Aug 21 2015 15:46:21 GMT+0800 (中国标准时间)

```

而`new Date()`则是会根据参数来返回对应的值，无参数的时候，返回当前时间的字符串形式；有参数的时候返回参数所对应时间的字符串。`new Date()`对参数不管是格式还是内容都要求，且只返回字符串。

```js

new Date();
//Fri Aug 21 2015 15:51:55 GMT+0800 (中国标准时间)

new Date(1293879600000);
new Date('2011-01-01T11:00:00')
new Date('2011/01/01 11:00:00')
new Date(2011,0,1,11,0,0)
new Date('jan 01 2011,11 11:00:00')
new Date('Sat Jan 01 2011 11:00:00')
//Sat Jan 01 2011 11:00:00 GMT+0800 (中国标准时间)

new Date('sss');
new Date('2011/01/01T11:00:00');
new Date('2011-01-01-11:00:00')
new Date('1293879600000');
//Invalid Date

new Date('2011-01-01T11:00:00')-new Date('1992/02/11 12:00:12')
//596069988000

```

从上面的几个测试结果可以很容易发现：

1.	`new Date()`在参数正常的情况只会返回当前时间的字符串（且是当前时区的时间）
2.	`new Date()`在解析一个具体的时间的时候，对参数有较严格的格式要求，格式不正确的时候会直接返回`Invalid Date`，比如将`number`类的时间戳转换成`string`类的时候会导致解析出错
3.	虽然`new Date()`的返回值是字符串，然而两个`new Date()`的结果字符串是可以直接相减的，结果为结果为毫秒数。

那么，`new Date()`能接受的参数格式到底是什么标准呢？（相对于严格的多参数传方法。非严格的单参数（数字日期表示格式）更常用且更容易出错，所以下文只考虑单参数数字时间字符串转换的情况）

## new Date()解析所有支持的参数格式标准

### 时间戳格式

这个是最简单的也是最容易出错的。当然唯一的缺点大概就是对开发者不直观，无法一眼就看出具体日期。需要注意的一下两点：

> 1. js内的时间戳是指当前时间到`1970年1月1日00:00:00 UTC`对应的**毫秒数**，和unix时间戳不是一个概念，后者表示描述，差了1000倍
> 2. `new Date(timestamp)`中的时间戳必须是`number`格式，`string`会返回`Inalid Date`。所以比如`new Date('1111111')`这种写法是错误的

### 时间数字字符串格式

不大清楚这种改怎么描述，就是类似`YYYY/MM/DD HH:mm:ss`这种。下文以`dateString`代指。`new Date(dateString)`所支持的字符串格式需要满足[RFC2822标准](http://tools.ietf.org/html/rfc2822#page-14)或者[ISO 8601标准](http://www.w3.org/TR/NOTE-datetime)这两种标准对应的格式分别如下：

1.	RFC2822标准日起字符串

```js

YYYY/MM/DD HH:MM:SS ± timezon(时区用4位数字表示)
// eg 1992/02/12 12:23:22+0800

```

> RFC2822还有别的格式，不过上面这个是比较常用的（另外这个标准实在太难啃，实在没耐心啃完，所以也就没太深入）。RFC2822标准本身还有其他的非数字日起表达个事，不过不在这个话题讨论范围内，略过。

2.	ISO 8601标准日起字符串

```js

 YYYY-MM-DDThh:mm:ss ± timezone(时区用HH:MM表示)

 1997-07-16T08:20:30Z
 // “Z”表示UTC标准时区，即"00:00",所以这里表示零时区的`1997年7月16日08时20分30秒`
 //转换成位于东八区的北京时间则为`1997年7月17日16时20分30秒`

 1997-07-16T19:20:30+01:00
 // 表示东一区的1997年7月16日19时20秒30分，转换成UTC标准时间的话是1997-07-16T18:20:30Z

```

> 1. 日期和事件中间的`T`不可以省略，一省略就出错
> 2. 虽然在chrome浏览器上失去也可以用`+0100`这种RFC2822形式，然而IE上不支持这种混搭写法，所以用ISO8601标准形式表示的时候时区要用`+HH:MM`

淡淡从个事上来说，两者的区别主要在分隔符的不同。不过需要注意的是，ISO8601标准的兼容性比RFC2822差很多（比如IE8和ios均不支持前者）。所以一般情况下建议用`RFC2822`格式的。

不过需要注意的是，在未指定时区的前提下，对于只精确带`day`的日起字符串，`RFC2822`返回结果是以`当前时区的零点`为准，而`iso8601`返回结果则会以`UTC时间`的零点为标准进行解析。

例如：

```js

//RFC2822：
new Date('1992/02/13') //Thu Feb 13 1992 00:00:00 GMT+0800 (中国标准时间)
//ISO8601:
new Date('1992-02-13') //Thu Feb 13 1992 08:00:00 GMT+0800 (中国标准时间)

```

然而上面这个只是ES5的标准而已，在ES6里这两种形式都会变成`当前时区的零点`为基准。

关于跨浏览器的dataString解析情况，还可以参考这个页面：[http://dygraphs.com/date-formats.html](http://dygraphs.com/date-formats.html)

**所以对于时间字符串对象，个人意见是要么用`RFC2822`形式，要么自己写个解析函数然后随便你传啥格式进来**

## 时间格式化效率函数

这里的`时间格式化`指的是将时间字符串传换成毫秒的过程。js原生的时间格式化函数有`Date.parse`、`Date.prototype.valueOf`、`Date.prototype.getTime`、`Number(Date)`、`+Date`（还有个`Date.UTC`方法，然而对参数要求严格，不能直接解析日期字符串，所以略过）

这五个函数从功能上来说一模一样，但是在具体效率如何？我写了一个检测页面，诸位也可以自己测试一下[http://codepen.io/chitanda/pen/NqeZag/](http://codepen.io/chitanda/pen/NqeZag/)

### 核心测试函数：

```javascript

function test(dateString,times,func){
	var startTime=window.performance.now();
	// console.log('start='+startTime.getTime());
	for (var i = 0; i < times; i++) {
	    func(dateString);//这里填写具体的解析函数
	};
	var endTime=window.performance.now();
	// console.log('endTime='+endTime.getTime());
	var gapTime=endTime-startTime;
	  console.log('一共耗时:'+gapTime+'ms');
	// console.log('时间字符串'+dateString);
	return gapTime;
}

```

> 之所以这里用`window.performance.now()`而不用`new Date()`，是因为前者精准度远比后者高。后者只能精确到ms。会对结果造成较大的影响

### 测试结果

单次执行50W次时间格式化函数，并重复测试100次，最后的结果如下：（表格中的数字为单次执行50W次函数的平均结果。单位为毫秒）

<table>
	<tr>
		<th>函数</th>
		<th>Chrome</th>
		<th>IE</th>
		<th>Firefox</th>
	</tr>
	<tr>
		<td>Date.parse()</td>
		<td>151.2087</td>
		<td>55.5811</td>
		<td>315.0446</td>
	</tr>
	<tr>
		<td>Date.prototype.getTime()</td>
		<td>19.5452</td>
		<td>21.3423</td>
		<td>14.0169</td>
	</tr>
	<tr>
		<td>Date.prototype.valueOf()</td>
		<td>20.1696</td>
		<td>21.7192</td>
		<td>13.8096</td>
	</tr>
	<tr>
		<td>+Date()</td>
		<td>20.0044</td>
		<td>31.3511</td>
		<td>22.7861</td>
	</tr>
	<tr>
		<td>Number(Date)</td>
		<td>23.0900</td>
		<td>24.8838</td>
		<td>23.3775</td>
	</tr>
</table>

从这个表格可以很容易得出以下结论：

> 1. 从计算效率上来说：`Date.prototype.getTime()≈Date.prototype.valueOf()>+Date≈Number(Date)>>Date.parse()`
> 2. 从代码书写效率上来说，对于少量的时间格式化计算，用`+Date()`或者`Number(Date)`即可。而若页面内有大量该处理，则建议用Date原生的函数`Date.prototype.getTime()`或者`Date.prototype.valueOf()`只有`Date.parse`，找不到任何使用的理由。
> 3. 这个结果和计算机的计算性能以及浏览器有关，所以具体数字可能有较大偏差，很正常。然而几个函数结果的时间差大小顺序不会变。
> 4. codepen的在线demo显示比较大，对于这个测试个人建议最好吗源码赋值到本地文件然后进行测试。

## UTC，GMT时间的区别

这个不是啥重要的东西，单纯当课外知识吧

### 格林威治标准时间GMT

GMT即【格林威治标准时间】（Dreenwich Mean Time，简称G.M.T.），指位于英国伦敦郊区的皇家格林威治天文台的标准时间，因为本初子午线被定义为通过哪里的经线。**然而由于地球的不规则自传，导致GMT时间有误差，因此目前不被当做标准时间使用**。

### 时间协调时间UTC

UTC是最主要的世界时间标准，是经过平均太阳时（以格林威治时间GMT为准），地轴运动修正后的新时标准以及以【秒】为单位的国际原子能时所综合精算而成的时间。UTC比GMT来的更加精准。其误差值必须保持在0.9秒以内，若大于0.9秒则由位于巴黎的国际第七自转事务中央局发布闰秒，使用UTC与地球自转周期一直。**不过日常使用中，GMT和UTC的功能与精准度是没有差别的**。

协调世界时区会使用“Z”来表示。而在航空上，所有使用的时间统一规定是协调世界时。而且Z在无线电中应读作“Zulu”（可参考北约音标字母），协调世界时也会被称为“Zulu time”。

## 浏览器回去用户当前时间以及洗好语言

首先需要注意的一点，浏览器获取当前用户所在的失去等信息只和系统的`日期和时间`设置里的时区以及时间有关。`区域和语言`设置影响的是浏览器默认时间函数（Date.prototype.toLocaleString）显示格式，不会对时区等有影响。以Windows为例，`控制面板\时钟、语言和区域`中的两个子设置项目的区别如下：

> `日期和时间`：设置当前用户所处的时间和时区，浏览器获取的结果以此为准，哪怕用户的设置时间和时区是完全错误的。比如若东八区的用户将自己的时区设置为东九区，浏览器就会将视为东九区；时间数据上同理。这里的设置会影响`Date.prototype.getTimezoneOffset、 new Date()`的值

`区域和语言`：主要是设置系统默认的时间显示方式。其子设置的`格式`会影响`Date.prototype.toLocaleString`方法返回的字符串结果。

## 浏览器判断用户本地字符串格式

Date有个`Date.prototype.toLocaleString()`方法可以将时间字符串返回用户本地字符串格式，这个发发还有两个子方法`Date.prototype.toLocaleDateString`和`Date.prototype.toLocaleTimeString`，这两个方法返回值分别表示`日期`和`时间`，加在一起就是`Date.prototype.toLocaleString`的结果。

这个方法的默认参数会对时间字符串做一次转换，将其转换为用户当前所在时区的时间，并按照对应的系统设置时间格式返回字符串结果。**然而不同浏览器对用户本地所使用的语言格式的判断依据是不同的**。

IE：获取系统当前的`区域和语言`-`格式`格式，只按照自己浏览器的结果设置的菜单语言来处理。（比如英文界面则按系统‘en-US’格式返回字符串，中文界面则按照系统‘zh-CN’格式返回结果）综上可得：

> Chrome下**浏览器语言设置优先系统语言设置**。而IE和FF则是**系统语言设置优先浏览器语言设置**，不管浏览器界面语言是什么，他们只依照系统设置来返回格式。

>另外，不同浏览器对`toLocalString`返回的结果也是不同的，IE浏览器严格遵循系统设置，而Chrome和FF会有自己内置的格式来替换。


### 浏览器界面语言设置和语言设置的区别

这小节貌似有点跑题，然而不说明下的很容易和上面提到的浏览器设置的语言混淆，所以也拿出来说一下。

需要注意**浏览器的语言设置和界面语言设置不是一回事**。

浏览器的语言设置设置的是浏览器发送给服务器的Request Header里的Accept-Language的值，这个值可以告诉服务器用户的喜好语言，对于某些跨国网站，服务器可以以此为依旧来返回对应语言的页面（不过实际应用上这个限制比较大，大部分网站还是根据IP来判断用户来源的，或者直接让用户自己选择）

对于各大浏览器而言，这个设置的更改也是比较显性，容易找到的。

IE： Internet选项-语言

FF： 选项-内容-语言

chrome:设置-显示高级设置-语言-语言和输入设置...

上面这里的设置不会影响到浏览器的界面语言设置，以国内大部分用户而言，即不管你怎么设置这里的语言选项，浏览器菜单等默认都会是以中文显示的.

而浏览器的界面语言设置一般来说则藏的深得多，没那么容易找到。

IE：
卸载前面安装过的浏览器语言包，去微软官网下载对应的IE浏览器语言包安装。（和安装的语言包有关。系统界面语言和该语言包相同的情况下，变为该语言。否则以安装的语言包为准。）

FF：地址栏输入about:config，然后找到general.useragent.locale字段，修改对应字段即可。
chrome：设置-显示高级设置-语言-语言和输入设置...

## 利用JS获取用户浏览器语言喜好

对于获取这两种设置，js原声方法支持度比较一般：

IE下的`navigator`方法有四种和`language`有关的方法，区别如下：

假设系统语言为`ja-JP`，系统unicode语言为`zh-CN`日起格式为`nl-NL`浏览器语言设置（accept-language）为`de`，浏览器界面语言为`en-US`（其他条件不变，浏览器界面语言改为`zh-CN`的时候结果也是一样）

```js

window.navigator.language
//"nl-NL"
window.navigator.systemLanguage
//"zh-CN"(设置中的非unicode程序所使用语言选项)
window.navigator.userLanguage
//"nl-NL"
window.navigator.browserLanguage
//"ja-JP"（系统菜单界面语言）
window.navigator.languages
//undefined

```

chrome下，当浏览器界面语言为`zh-CN`，`accept-language`首位为`en-US`的时候：

```js

window.navigator.language
//'en-US'
window.navigator.languages
//["en-US", "zh-CN", "de", "zh", "en"]
//当界面语言改为"en-US",`accept-language`首位为`zh-CN`的时候
window.navigator.language
//'zh-CN'（`accept-language`首选值)
window.navigator.languages
//["zh-CN", "de", "zh", "en-US", "en"]

```

> 1. 从上面的测试结果可以很明显的发现IE浏览器的这几个函数都是获取系统信息的，无法获取当前提到的两个浏览器层面上的设置（这几个函数具体含义还有疑问的可以参考[MDN官方文档](https://msdn.microsoft.com/en-us/library/ms535867.aspx)）
> 2. `window.navigator.language`这个函数虽然三个浏览器都可以兼容，然而代表的意义完全不同。IE下该函数返回系统设置的事件显示个是所遵守的标准的地区代码；chrome下返回浏览器界面语言；FF下返回`accept-language`的首选语言值

由此：

> 1.**浏览器设置的语言**即`accept-language`值，IE浏览器无法用JS获取。chrome和FF浏览器都可以利用`window.navigator.languages`来获取，而FF还可以直接用`window.navigator.language`直接获取`accept-language`的首选语言值。所以`accept-language`，兼容性最好的回去方法应该是利用后端，发起一个ajax请求，分析header。而不是直接js来处理。
> 2.**浏览器界面语言**，IE和FF都无法利用js来获取，chrome可以用`window.navigator.language`来获取
> 3. 系统级别的语言设置（系统菜单界面语言，系统设置的时间显示格式），chrome和ff都无法用js获取到

## 参考文献

1.	[Date and Time Formats](http://www.w3.org/TR/NOTE-datetime)
2.	[Date and Time Specification(RFC2822)](http://tools.ietf.org/html/rfc2822#page-14)
3.	[Date.parse()-Differences in assumed time zone](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/parse#Differences_in_assumed_time_zone)
4.	[JavaScript and Dates, What a Mess!](http://blog.dygraphs.com/2012/03/javascript-and-dates-what-mess.html)
5.	[navigator object(IE浏览器私有language函数的解析)](https://msdn.microsoft.com/en-us/library/ms535867.aspx)

转自：[JS原生Date类型方法的一些冷知识](https://segmentfault.com/a/1190000003710954)

**smileyby**觉得前面的理解起来还好，后面的语言部分稍显逻辑混乱，云里雾里！！


