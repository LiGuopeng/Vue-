##Vue双向绑定数据原理 是  通过数据劫持结合发布者-订阅者模式来实现的.

####1:首先看个例子
     
			var vm = new Vue({
			    data: {
			        obj: {
			            a: 1
			        }
			    },
			    created: function () {
			        console.log(this.obj);
			    }
			});
 接下里我们可以具体观看obj的对象里面有哪些属性(手撸去看  >>明显有 get ,set)方法

####a>>>>那么get,set 方法是从来过来的呢？>>>肯定在操作对象的方法内

        vue通过有defineProperty来实现数据劫持的。 defineProperty是用来干啥的？

        它可以来控制一个对象属性的一些特有操作，比如读写权、是否可以枚举，这里我们主要先来研究下它对应的两个描述属性get和set
eg:来看下面的example  
      
		     var Book = {
				  name: 'vue权威指南'
				};
				console.log(Book.name);  // vue权威指南
如果想要在Book.name值过程中去改变这个值，如何处理？通过什么监听对象 Book 的属性值。这时候Object.defineProperty( )就派上用场了
        var Book={}
        Object.defineProperty(Book,'name'){
		  set: function (value) {
		    name = value;
		    console.log('你取了一个书名叫做' + value);
		  },
		  get: function () {
		    return '《' + name + '》'
		  }
		})
 
			Book.name = 'vue权威指南';  // 你取了一个书名叫做vue权威指南
			console.log(Book.name);  // 《vue权威指南》
由此案例我们可以看出  get就是读取属性的，set就是设置这个属性触发的函数



##2通过以上例子 我们来大体分析一下
###2>1  我们如何知道数据发生变化（关键点）（大概我们可以猜到必须用get,set这两个方法）   >>之前我们提到set(Object.defineProperty( )对属性设置一个set函数设置数据的)>>set只是对改变的数据赋值>>那我们是如何对比数据发生变化调取set呢（大概的话我们比较数据发生变化时>>必定有get取值这一步骤）


###3：接下来我们下理一下过程，再具体分析在 get ,set 中我们进行了那些逻辑操作
        
        过程：1：首先要对数据进行劫持监听，所以我们需要一个Observer监听器,用来监听属性

             2：如果属性发生变化，我们则会告诉订阅者（Watch）看是否更新

             3：因为订阅者有多个，所以我们需要有个消息订阅者（Dept）专门去收集订阅者

             4：接下来我们在监听器（Observer）与订阅者（Watch）进行统一的管理
  
             5：涉及到指令解析器（Compile）,对每个节点的元素进行扫描和解析，将有关指令初始化成一个Watch订阅者，并替换模板和绑定对应的函数，此时当订阅者Watcher接收到相应属性的变化，就会执行对应的更新函数，从而更新视图

网上找了个总结比较好的

        1.实现一个监听器Observer，用来劫持并监听所有属性，如果有变动的，就通知订阅者。
		
		2.实现一个订阅者Watcher，可以收到属性的变化通知并执行相应的函数，从而更新视图。
		
		3.实现一个解析器Compile，可以扫描和解析每个节点的相关指令，并根据初始化模板数据以及初始化相应的订阅器。

##3 那我们按上面的步骤走
###3-1 监听器Observer  (监听属性，必须把全部的属性遍历出来>>递归)
      Observer.js

      funcation observe（data）{
      if(!data ||typeof data！=='object'){return}
      Object.keys(data).forEach(funcation(key){
        //递归一个对象>>相当于降维
       defineReactive(data, key, data[key]);

      })

     function defineReactive(data, key, val) {
         observe(val); // 递归遍历所有子属性
      Object.defineProperty(data,key,{
         enumerable: true,
        configurable: true,
        get:function(){return val}
        set:funcation（newval）{
           val=newval
       console.log('属性' + key + '已经被监听了，现在值为：“' + newVal.toString() + '”');

       }
     })     
		var library = {
		    book1: {
		        name: ''
		    },
		    book2: ''
		};
		observe(library);
      }  
     }

###3-2  既然已经监听完属性>>既然属性改变时，我们则会告诉订阅者（Watch），接下来我们对订阅者（Watch）进行操作(涉及到订阅者有多个>>dept(消息订阅器)负责收集订阅，然后再属性变化的时候执行对应订阅者的更新函数)   显然dept 相当于容器.>>>也可以想象成数组 

######订阅器  相当于  订阅者   的父集   啊（也可以这样去了解）

####3-2-1 我们先按上边的思想去列代码，之后再整合    

         1：首先我们定义一个订阅器
          function Dept(){this.subs=[]}

         2：初始化对象
          var dept=new Dept();
       
         3：当检测到数据变换时,我们需要将watch  添加到dept中>>当然在订阅器中的原型上定义增加的方法
          
         Dept.prototype={
            addSub: function(sub) {
            this.subs.push(sub);
           },

       }
       4：既然数据发生了变化>>我们则去通知所有订阅者>>当然也是在订阅器中的原型中定义此方法
        Dept.prototype={

             notify: function() {
             this.subs.forEach(function(sub) {
            sub.update();
        });

       }
####3-2-2 如上我们大体整理出来思想后，接下来我们修改 Observer.js文件
       
           //定义Dept订阅器对象
			      function Dept () {
			    this.subs = [];
			}
           //增加其方法
			Dept.prototype = {
			    addSub: function(sub) {
			        this.subs.push(sub);
			    },
			    notify: function() {
			        this.subs.forEach(function(sub) {
			            sub.update();  //watch触发
			        });
			    }
			};

      funcation observe（data）{
      if(!data ||typeof data！=='object'){return}
      Object.keys(data).forEach(funcation(key){
        //递归一个对象>>相当于降维
       defineReactive(data, key, data[key]);

      })

     function defineReactive(data, key, val) {
         observe(val); // 递归遍历所有子属性
         var Dept = new Dept(); 
        Object.defineProperty(data,key,{
         enumerable: true,
        configurable: true,
        get:function(){
         if (是否需要添加订阅者) {
                Dept.addSub(watcher); // 在这里添加一个订阅者
            }
            return val;

       }
        set:funcation（newval）{
           if（val=newval）{
             return;
            }
           val=newval
       console.log('属性' + key + '已经被监听了，现在值为：“' + newVal.toString() + '”');
        dept.notify(); // 如果数据变化，通知所有订阅者

       }
     })     
		var library = {
		    book1: {
		        name: ''
		    },
		    book2: ''
		};
		observe(library);
      }  
     }

##实现Watch 
###3-3  订阅者在初始化的时候如何将自己添加到订阅器Dept 中>>>>我们已经知道监听器Observer是在get函数执行了添加订阅者Wather的操作的，所以我们只要在订阅者Watcher初始化的时候出发对应的get函数去执行添加订阅者操作即可，那要如何触发get的函数

###核心如何触发get呢>>>因为我们使用了Object.defineProperty( )进行数据监听。>>>>>>>>>>>这里还有一个细节点需要处理，我们只要在订阅者Watcher初始化的时候才需要添加订阅者，所以需要做一个判断操作，因此可以在订阅器上做一下手脚：在Dept.target上缓存下订阅者，添加成功后再将其去掉就可以了。订阅者Watcher的实现如下：

Watch.js

     funcation Watcher(vm,exp,cp){

			    this.cb = cb; //调用$watch时候传进来的回调
			    this.vm = vm;
			    this.exp = exp; //这里的expOrFn是你要监听的属性或方法也就是$watch方法的第一个参数（为了简单起见，我们这里补考录方法，只考虑单个属性的监听）
			    this.value = this.get();  // 将自己添加到订阅器的操作

     }
  get方法  Watch初始化是如何添加进去的呢
	
	    Watcher.prototype={
	       update: function() {// 还记得Dep.notify方法里循环的update么？
	        this.run();
	    },
	     run(){//这个方法并不是实例化Watcher的时候执行的，而是监听的变量变化的时候才执行的
	    var value = this.vm.data[this.exp];
        var oldVal = this.value;
        if (value !== oldVal) {
            this.value = value;
            this.cb.call(this.vm, value, oldVal);
        }
	     }
	    get:function(){
	      Dept.target=this;  //将Dep身上的target 赋值为Watcher对象
	      var value=this.vm.data.[this.exp] //这里拿到你要监听的值，在变化之前的数值
	    (声明value，使用this.vm._data进行赋值，并且触发_data[a]的get事件)
	      Dept.target=null      //释放自己
	       return  value;
	    }
         }

	   在吧Watcher对象放再Dep.subs数组中之后，new Watcher对象所执行的任务就告一段落，此时我们有：
	
	　 1.Dep.subs数组中，已经添加了一个Watcher对象，
	
	　 2.Dep对象身上有notify方法，来触发subs队列中的Watcher的update方法，
	
	　 3.Watcher对象身上有update方法可以调用run方法可以触发最终我们传进去的回调



接下来改动Observer 的判断代码

             if (Dep.target) {.  // 判断是否需要添加订阅者
                dep.addSub(Dep.target); // 在这里添加一个订阅者
            }
            return val;


	实现Compile
	虽然上面已经实现了一个双向数据绑定的例子，但是整个过程都没有去解析dom节点，而是直接固定某个节点进行替换数据的，所以接下来需要实现一个解析器Compile来做解析和绑定工作。解析器Compile实现步骤：
	
	1.解析模板指令，并替换模板数据，初始化视图
	
	2.将模板指令对应的节点绑定对应的更新函数，初始化相应的订阅器
	
	为了解析模板，首先需要获取到dom元素，然后对含有dom元素上含有指令的节点进行处理，因此这个环节需要对dom操作比较频繁，所有可以先建一个fragment片段，将需要解析的dom节点存入fragment片段里再进行处理：
	
	function nodeToFragment (el) {
	    var fragment = document.createDocumentFragment();
	    var child = el.firstChild;
	    while (child) {
	        // 将Dom元素移入fragment中
	        fragment.appendChild(child);
	        child = el.firstChild
	    }
	    return fragment;
	}
	
	接下来需要遍历各个节点，对含有相关指定的节点进行特殊处理，这里咱们先处理最简单的情况，只对带有 '{{变量}}' 这种形式的指令进行处理，先简道难嘛，后面再考虑更多指令情况：
	
	function compileElement (el) {
	    var childNodes = el.childNodes;
	    var self = this;
	    [].slice.call(childNodes).forEach(function(node) {
	        var reg = /\{\{(.*)\}\}/;
	        var text = node.textContent;
	 
	        if (self.isTextNode(node) && reg.test(text)) {  // 判断是否是符合这种形式{{}}的指令
	            self.compileText(node, reg.exec(text)[1]);
	        }
	 
	        if (node.childNodes && node.childNodes.length) {
	            self.compileElement(node);  // 继续递归遍历子节点
	        }
	    });
	},
	function compileText (node, exp) {
	    var self = this;
	    var initText = this.vm[exp];
	    this.updateText(node, initText);  // 将初始化的数据初始化到视图中
	    new Watcher(this.vm, exp, function (value) {  // 生成订阅器并绑定更新函数
	        self.updateText(node, value);
	    });
	},
	function (node, value) {
	    node.textContent = typeof value == 'undefined' ? '' : value;
	}

	获取到最外层节点后，调用compileElement函数，对所有子节点进行判断，如果节点是文本节点且匹配{{}}这种形式指令的节点就开始进行编译处理，编译处理首先需要初始化视图数据，对应上面所说的步骤1，接下去需要生成一个并绑定更新函数的订阅器，对应上面所说的步骤2。这样就完成指令的解析、初始化、编译三个过程，一个解析器Compile也就可以正常的工作了。为了将解析器Compile与监听器Observer和订阅者Watcher关联起来，我们需要再修改一下类SelfVue函数：
	
	
	function SelfVue (options) {
	    var self = this;
	    this.vm = this;
	    this.data = options;
	 
	    Object.keys(this.data).forEach(function(key) {
	        self.proxyKeys(key);
	    });
	 
	    observe(this.data);
	    new Compile(options, this.vm);
	    return this;
	}
	
	更改后，我们就不要像之前通过传入固定的元素值进行双向绑定了，可以随便命名各种变量进行双向绑定了：

	<body>
	    <div id="app">
	        <h2>{{title}}</h2>
	        <h1>{{name}}</h1>
	    </div>
	</body>
	<script src="js/observer.js"></script>
	<script src="js/watcher.js"></script>
	<script src="js/compile.js"></script>
	<script src="js/index.js"></script>
	<script type="text/javascript">
 
    var selfVue = new SelfVue({
        el: '#app',
        data: {
            title: 'hello world',
            name: ''
        }
    });
 
    window.setTimeout(function () {
        selfVue.title = '你好';
    }, 2000);
 
    window.setTimeout(function () {
        selfVue.name = 'canfoo';
    }, 2500);
 
	</script>
	
	如上代码，在页面上可观察到，刚开始titile和name分别被初始化为 'hello world' 和空，2s后title被替换成 '你好' 3s后name被替换成 'canfoo' 了。废话不多说，再给你们来一个这个版本的代码（v2），获取代码！
	
	到这里，一个数据双向绑定功能已经基本完成了，接下去就是需要完善更多指令的解析编译，在哪里进行更多指令的处理呢？答案很明显，只要在上文说的compileElement函数加上对其他指令节点进行判断，然后遍历其所有属性，看是否有匹配的指令的属性，如果有的话，就对其进行解析编译。这里我们再添加一个v-model指令和事件指令的解析编译，对于这些节点我们使用函数compile进行解析处理：


	function compile (node) {
	    var nodeAttrs = node.attributes;
	    var self = this;
	    Array.prototype.forEach.call(nodeAttrs, function(attr) {
	        var attrName = attr.name;
	        if (self.isDirective(attrName)) {
	            var exp = attr.value;
	            var dir = attrName.substring(2);
	            if (self.isEventDirective(dir)) {  // 事件指令
	                self.compileEvent(node, self.vm, exp, dir);
	            } else {  // v-model 指令
	                self.compileModel(node, self.vm, exp, dir);
	            }
	            node.removeAttribute(attrName);
	        }
	    });
	}
	
	上面的compile函数是挂载Compile原型上的，它首先遍历所有节点属性，然后再判断属性是否是指令属性，如果是的话再区分是哪种指令，再进行相应的处理，处理方法相对来说比较简单，这里就不再列出来，想要马上看阅读代码的同学可以马上点击这里获取。
	
	最后我们在稍微改造下类SelfVue，使它更像vue的用法：
	
	
	function SelfVue (options) {
    var self = this;
    this.data = options.data;
    this.methods = options.methods;
 
    Object.keys(this.data).forEach(function(key) {
        self.proxyKeys(key);
    });
 
    observe(this.data);
    new Compile(options.el, this);
    options.mounted.call(this); // 所有事情处理好后执行mounted函数
}

		这时候我们可以来真正测试了，在页面上设置如下东西：


	<body>
	    <div id="app">
	        <h2>{{title}}</h2>
	        <input v-model="name">
	        <h1>{{name}}</h1>
	        <button v-on:click="clickMe">click me!</button>
	    </div>
	</body>
	<script src="js/observer.js"></script>
	<script src="js/watcher.js"></script>
	<script src="js/compile.js"></script>
	<script src="js/index.js"></script>
	<script type="text/javascript">
	 
	     new SelfVue({
	        el: '#app',
	        data: {
	            title: 'hello world',
	            name: 'canfoo'
	        },
	        methods: {
	            clickMe: function () {
	                this.title = 'hello world';
	            }
	        },
	        mounted: function () {
	            window.setTimeout(() => {
	                this.title = '你好';
	            }, 1000);
	        }
	    });
	 
	</script>
	
	
	       




      

          
        