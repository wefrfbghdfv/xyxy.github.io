XSS（Cross-Site Scripting，跨站脚本）漏洞是一种常见的Web安全漏洞，攻击者通过在Web页面中注入恶意脚本代码，使得这些脚本在其他用户浏览器中执行，从而窃取用户信息、篡改页面内容、执行恶意操作等。

### XSS漏洞的类型

1. **反射型XSS（Reflected XSS）**：

   - 攻击者将恶意脚本嵌入到请求的URL中（如查询字符串、HTTP头等），当用户访问这个URL时，脚本会被立即执行。通常这种攻击通过社会工程学手段诱导用户点击恶意链接。

   - 示例：

     ```
     php-template
     
     
     复制编辑
     https://example.com/search?q=<script>alert('XSS')</script>
     ```

2. **存储型XSS（Stored XSS）**：

   - 攻击者将恶意脚本提交到服务器（例如通过表单、评论、输入框等），并将这些脚本永久存储在服务器端。然后，当其他用户访问含有恶意脚本的页面时，这些脚本会自动执行。
   - 存储型XSS相对更危险，因为它不依赖用户的交互，脚本一旦存储成功，所有访问该页面的用户都会受到攻击。
   - 示例：在一个评论区提交 `<script>alert('XSS')</script>`，并存储在数据库中，其他用户查看评论时会执行这个脚本。

3. **DOM型XSS（DOM-based XSS）**：

   - 这种攻击利用客户端脚本（例如JavaScript）动态修改页面内容。攻击者通过篡改页面中的DOM结构，导致脚本执行。不同于反射型XSS和存储型XSS，DOM型XSS是在浏览器端直接执行的，攻击不依赖于服务器端的响应。
   - 示例：攻击者可能利用一个页面中错误的JavaScript代码，在页面加载时通过JavaScript直接操作URL参数或其他不安全的内容，注入恶意脚本。

1. xss可以使用HTML实体编码、urlencoed 、js unicode编码进行编码绕过。而浏览器先解析HTML实体编码，在解析urlencode编码，最后解析js  unicode编码

2. 可以通过HTML实体编码进行反射性XSS的攻击

3. 可以使用a标签进行绕过

4. 可以使用JavaScript的伪协议绕过

5. 可以使用autofocus+onfocus进行自动聚焦，不使用于参与即刻触发

6. DOM破坏通过接收的参数名进行传参，通过一定的绕过方式，

   - [ ] 例如：eval(alert(1))

   - [ ] 双引号闭合绕过

   - [ ] 使用//注释掉后面代码进行绕过

   - [ ] 通过进入标签开始状态，解析为一个完整的token值绕过

     

# JavaScript 原型链污染

原型链污染是指攻击者通过操控对象的原型链，使得对象继承不应有的属性或方法，从而影响程序的行为。攻击者可能通过修改对象的原型链，继承恶意属性，破坏程序的正常逻辑或引发安全漏洞。

在JavaScript中，每个对象都有一个 `prototype` 属性，指向其构造函数的原型对象。通过原型链，子对象可以继承父对象的属性和方法。

例如，`obj.__proto__ = SomeConstructor.prototype`。

**修改 `Object.prototype`**：

- 攻击者可以直接修改 `Object.prototype`，使得所有的普通对象（`{}`）都继承这个恶意属性或方法。

- 例如：

  ```
  js复制编辑Object.prototype.isHacked = true;
  const obj = {};
  console.log(obj.isHacked); // 输出 true
  ```

**修改构造函数的原型**：

- 通过修改某个构造函数的原型对象，攻击者可以使得该构造函数创建的所有实例都受到影响。

- 例如：

  ```
  js复制编辑function MyClass() {}
  MyClass.prototype.isHacked = true;
  const obj = new MyClass();
  console.log(obj.isHacked); // 输出 true
  ```

**通过 `__proto__` 直接操控原型链**：

- `__proto__` 允许访问对象的原型，攻击者可以直接修改它来篡改对象的原型链。

- 例如：

  ```
  js复制编辑const obj = {};
  obj.__proto__.isHacked = true;
  console.log(obj.isHacked); // 输出 true
  ```

###  **XSS原型链污染**

原型链污染（Prototype Chain Pollution）是 **XSS** 攻击中的一种特殊变种，指的是攻击者通过注入恶意脚本，操控对象的原型链，使得该对象或其他对象继承不应有的属性或方法。

在JavaScript中，所有对象都继承自 `Object.prototype`。每当你创建一个新的对象时，这个对象的原型会指向 `Object.prototype`，而这个原型可以被修改

可以通过修改 `Object.prototype` 或某些对象的 `__proto__`。进行污染