#五分钟速成JavaScript

JavaScript是一门动态类型的语言，JavaScript的类型包括

1. undefined 未申明，或者变量的值即为undefined或者未初始化

	```javascript
	// a未定义
	typeof a
	```

1. boolean 布尔类型

	```javascript
	// 定义a
	var a = true
	typeof a
	```

1. string  字符串类型
	
	```javascript
	// 动态类型
	a = "true"
	typeof a
	```

1. number  数字类型，没有int类型

	```javascript
	a = 12
	typeof a
	// 输出
	console.log(a)
	```

1. object 对象或者值为null

	```javascript
	a = null
	typeof a

	a = {}
	typeof a

	a.weight = 20
	console.log(a)
	typeof JSON.stringify(a)
	console.log(JSON.stringify(a))

	a = []
	typeof a
	a[0] = 1
	a[1] = {
		attr1: "abc",
		attr2: [345]
	}
	console.log(JSON.stringify(a))
	```

1. function 函数

	```javascript
	a = function(temp) {
		for (var i = 0; i < 10; ++i) {
			console.log(i);
		}
		// JavaScript是函数作用域
		console.log("i" + i)
		console.log(temp)
	}
	typeof a
	a("于秀娟")

	// 函数是一等公民
	b = function(fuc) {
		fuc("wanwenkai")
	}

	b(a)

	```

Android调用

	```javascript
	androidAntLinker.GetAppVersion(JSON.stringify(request));
	```
	```java
	@JavascriptInterface
        public void GetAppVersion(String request) {
            Log.e("GetAppVersion", request);
            webView.post(new Runnable() {
                @Override
                public void run() {
                    String Version = "1.2.3";
                    webView.evaluateJavascript("antlinker.getAppVersion.success(new Object({Version: '" + Version + "'}))", null);
                }
            });
        }
        ```


