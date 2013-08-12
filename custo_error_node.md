#Node.js下自定义错误类型

在JavaScript里面，运行过程中的错误的类型总是被人忽略，这篇教程主要从三个方面来介绍如何在Node.js下自定义错误类型。

+ 为什么要使用错误对象。
+ 怎么创建自定义错误对象。
+ 一些自定义错误对象的例子。

##为什么要使用错误对象

一般来说，很少人会考虑如何处理应用产生的错误的策略，调试的过程中，简单地利用console.log('error')定位错误，基本够用了，通过留下这些调试信息，能够为我们以后的调试过程中升了不少时间，提高了维护性。所以错误提示非常重要。同时，也会带来一些比较糟糕用法。


###糟糕的字符串错误提示

Guillermo Rauch 曾经写了[一篇优秀的文章](http://www.devthought.com/2011/12/22/a-string-is-not-an-error/)解释为什么最好不要用字符串作为错误调信息，以下是一种常见的错误处理方式

  if (id < 0) return 'id must be positive'

直接返回错误字符串信息，虽然很简单，但是项目大了，就不好维护了，会产生以下这几个问题。

####很难知道错误在哪里发生的？

除非你的打印字符串里面，包含了文件和代码的行数，要不然，你根本不知道错误在哪里发生的。

####不能处理不同类型的错误

如果全部都是使用字符串作为错误提示，在运行的过程中，很难针对不同类型的错误进行不同的处理，有时候我们需要针对不同的错误进行不同类型的处理，如提交表单信息错误，数据库类型错误等。

##系统错误对象

利用系统自带的错误类型来代替纯字符串的错误提示

	if (id < 0) return new Error('id must be positive')

尽管看上去只是多了一点小步骤，但实际上却带了了很多好处。

###获取错误在哪里发生

如果用了错误，对象会产生堆栈属性，通过执行文件，通过行号，查找到出错的地方。
	
	console.log(error.stack)
	Error: An error occurred
	    at Object.<anonymous>(/Users/ds/main.js:1:73)
	    at Module._compile (module.js:441:26)
	    at Object..js (module.js:459:10)
	    at Module.load (module.js:348:31)
	    at Function._load (module.js:308:12)
	    at Array.0 (module.js:479:10)
	    at EventEmitter._tickCallback (node.js:192:40)

###获取错误类型

可以通过Instanceof 来检查错误类型，根据类型进行不同的处理

	if (err instanceof DatabaseError) { /* do this */ }
	if (err instanceof AuthenticationError) { /* do that */ }


##如何创建一个自定义的错误对象

通过对错误对象的继承与复写的手段，创建一个自定义的错误对象。

###抽象错误对象类型

创建一个抽象的错误类基类

	var util = require('util')
	
	var AbstractError = function (msg, constr) {
	  Error.captureStackTrace(this, constr || this)
	  this.message = msg || 'Error'
	}
	util.inherits(AbstractError, Error)
	AbstractError.prototype.name = 'Abstract Error'

###定义数据库错误对象

利用上述创建的抽象错误类型，扩展到其他自定义错误类型当中

	var DatabaseError = function (msg) {
	  DatabaseError.super_.call(this, msg, this.constructor)
	}
	util.inherits(DatabaseError, AbstractError)
	DatabaseError.prototype.message = 'Database Error'


错误类型部署到应用当中，下例

	function getUserById(id, callback) {
	  if (!id) {
	     return callback(new Error('Id is required'))
	  }
	  // Let’s pretend our database breaks if we try to
	  // find a user with an id higher than 10
	  if (id > 10) {
	    return callback(new DatabaseError(Id can't be higher ↵ than 10))
	  }
	  callback(null, { name: 'Harry Goldfarb' })
	}
	
	function onGetUserById(err, resp) {
	  if (err) {
	    return console.log(err.toString())
	  }
	  console.log('Success:', resp.name)
	}
	
	getUserById(1, onGetUserById) // Harry Goldfarb
	getUserById(null, onGetUserById) // Error: Id is required
	getUserById(53, onGetUserById) 
	// Database Error: Id can't be higher than 10

效果出来了，我们的getUserById 的方法，现在可以返回两种不同类型的错误，一个是原生的错误类型，一个是自定义的数据库错误类型databaseerror，如果我们调用toString，我们可以看到错误类型的返回


	// start our script in production mode
	$ NODE_ENV=production node main.js


	function onGetUserById(err, resp) {
	  if (err) {
	    if (err instanceof DatabaseError &&
	        process.env.NODE_ENV != 'production') {
	      return console.log(err)
	    }
	    return console.log('Sorry there was an error')
	  }
	  console.log(resp.name)
	}

根据不同错误类型进行不同情况的处理


##复用错误类型

我们不需要每次都重新定义我们的自定义错误在每一个文件里面，我们只需要创建一个文件，调用一个很重要的node.js方法require就足够了

	// in ApplicationErrors.js
	var util = require('util')
	
	var AbstractError = function (msg, constr) {
	  Error.captureStackTrace(this, constr || this)
	  this.message = msg || 'Error'
	}
	util.inherits(AbstractError, Error)
	AbstractError.prototype.name = 'Abstract Error'
	
	var DatabaseError = function (msg) {
	  DatabaseError.super_.call(this, msg, this.constructor)
	}
	util.inherits(DatabaseError, AbstractError)
	DatabaseError.prototype.name = 'Database Error'
	
	module.exports = {
	  Database: DatabaseError
	}
  

	// in main.js
	var ApplicationError = require('./ApplicationErrors')
	
	function getUserById(id, callback) {
	  if (!id) {
	    return callback(new Error('Id is required'))
	  }
	
	  if (id > 10) {
	    return callback(new ApplicationError.Database('Id cant ↵ be higher than 10'))
	  }
	
	  callback(null, { name: 'Harry Goldfarb' })
	}

由此可见，我们只需要定义一次，其他地方也能使用错误对象类型。

原文：[Using Custom Errors in Node.js](http://dustinsenos.com/articles/customErrorsInNode)
