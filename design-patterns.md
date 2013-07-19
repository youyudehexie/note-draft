#Node.js设计模式

##单例模式

**定义：**确保一个类只有一个实例，而且自行实例化并向整个系统提供这个实例。

单例模式根据实例化对象时机的不同分为两种：一种是饿汉式单例,一种是
懒汉式单例。饿汉式单例在单例类被加载时候，就实例化一个对象交给自己的引用；而懒汉式在调用取得实例方法的时候才会实例化对象。

###饿汉式单例
    var Person = function(){
      this.name = 'hexie';
    }
    
    Person.prototype.showName = function(){
    	return this.name;
    }
    
    var Singleton = function(){
    	var person = new Person()
    	this.getInstance = function(){
    		return person
    	}
    }

###懒汉式

	var Singleton = function(){
		var person = null;
		this.getInstance = function(){
			if(person === null){
				person = new Person();
			}
			return person;
		}
	}



###优点

+ 内存中只有一个对象，节省内存空间
+ 避免频繁销毁对象，提高性能。
+ 避免共享资源多重占用。
+ 可以全局访问。

##简单工厂模式

**定义**：定义一个用于创建对象的接口，让子类决定实例化哪一个类，工厂方法使一个类的实例化延迟到其子类。

	function Engine(){
		this.getStyle = '这是汽车的发动机';
	}
	
	function Underpan(){
		this.getStyle = '这是汽车的底盘';
	}
	
	function Wheel(){
		this.getStyle = '这是汽车的轮胎'
	}
	
	var ICar = function(underpan, wheel, engine){
	
		this.show = function(){
			[underpan, wheel, engine].forEach(function(style){
				console.log(style.getStyle);
			})
		}
	}
	
	var IFactory = function(){
		this.createCar = function(){
			var engine = new Engine();  
	    var underpan = new Underpan();  
	    var wheel = new Wheel();
	    var car = new ICar(underpan, wheel, engine);
	    car.show()
		}
	}
	
	var car = new IFactory().createCar()

**优点**：通过使用工厂方法而不是new关键字及具体类，可以把所有实例化的代码都集中在一个位置，有助于创建模块化的代码，这才是工厂模式的目的和优势。

##抽象工厂模式
抽象工厂模式的使用方法就是 - 先设计一个抽象类，这个类不能被实例化，只能用来派生子类，最后通过对子类的扩展实现工厂方法。

	var util = require('util');
	
	function Engine(){
		this.getStyle = '这是汽车的发动机';
	}
	
	function Underpan(){
		this.getStyle = '这是汽车的底盘';
	}
	
	function Wheel(){
		this.getStyle = '这是汽车的轮胎'
	}
	
	var IFactory = function(){};
	
	IFactory.prototype = {
		createFactory: function(){
			throw new Error('This is an abstract class');
		}
	}
	
	var Product = function(){
		IFactory.call(this);
	}
	
	util.inherits(Product, Product);
	
	Product.protype.createFactory = function(style){
		var result = null
		switch(style){
			case 'engine': result = new Engine(); break;
			case 'underpan': result = new Underpan(); break;
			case 'wheel': result = new Wheel(); break;
			default: result = new Engine(); break;
		}
		return result;
	
	}

##桥接模式

桥接方式主要是解耦，如果类直接都是充满各种相互调用的关系，这时候你就需要他了。

