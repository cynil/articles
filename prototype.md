# 从prototype到new

如何实现方法的复用呢？最容易想到的，就是：

```js
//mixin
function extend(optional, base){
    for(var prop in base){
        if(!prop in optional){
            optional[prop] = base[prop]
        }
    }
    
    return optional
}
```

这种方法俗称`mixin`，它直接从甲对象复制方法和属性到乙对象，乙对象就拥有了甲对象的功能，简单粗暴，易于理解。但是它有一个缺点，就是性能低下，浪费空间，因为每次扩展时都要复制一遍。那么，有没有更优雅的继承方法呢？

当然有。

实际上，`Javascript`天然内置了一种更优雅简单的方法，这就是人们耳熟能详的`prototype`原型链机制，该机制的内容可以描述为：

> 每一个对象内部都有一个`[[prototype]]`属性，该属性是一个指针，当对对象进行取值操作时，如果没有找到，则会尝试从该指针所指向的对象（即该对象的原型）中去查找，如果还没有找到，则会继续它的原型的原型中去查找，这个过程会一直持续下去，直到终点—— `Object.prototype`。

也可以图示为：

![原型链机制示意图](http://placeholder.exp)

这种对象扩展的方法十分清新，理论上说，如果想要乙对象从甲对象扩展方法，只要直接把乙对象的`[[prototype]]`属性指向甲对象即可。也就是说，`prototype`机制是一种在对象层面上的方法复用实现方式。而在经典的类式继承语言中，实现继承需要创建父类，然后创建子类继承父类，再实例化子类。

```
//pseudo code
class A{}
class B extends A{}
B foo = new B()
```
令人感到不可思议的是，虽然说原型机制是`Javascript`式继承的基石，但在`ECMAScript5`发布以前，`Javascript`竟然并没有提供可以把一个对象的`[[prototype]]`指针指向另一个对象的`API`，反而是用一种十分奇怪的方式模拟了典型类式语言的语法，也就是大家熟悉的构造函数和`new`。构造函数和`new`很好地满足了一些程序员对于经典类式语法的追求，但它掩盖了很多细节和原理，容易让人产生误解，这里暂且按下不表。

要实现纯`Javascript`原型式的继承，直接设置`[[prototype]]`指针的`API`是不可或缺的。很多（但不是所有）浏览器给对象暴露了一个可读可写的`__proto__`属性，可以用来设置对象的`[[prototype]]`指针，而`ES5`终于提供了一个`Object.create()`的方法，用于设置`[[prototype]]`指针。

使用`Object.create()`方法，可以轻松实现纯`Javascript`式继承，例如：

```js
var person = {
    speak: function(what){
        console.log(what)
    }
}
var artist = Object.create(person)

//I am an artist
artist.speak('I am an artist')
```

`artist`对象由`Object.create(person)`创建而来，因此它内部的`[[prototype]]`指针就指向`person`，当我们查询它的`foo`属性时，在它本身上找不到，于是就到它的原型上去查询。整个过程可以图示如下：

![person和artist的原型式继承](http://placeholder.exp)

这是一种轻量、简洁、易于理解的继承方式，我们可以随便创建新的`artist`对象，让它继承`person`，如果需要给所有的`artist`添加新的方法，只需在`person`添加就可立即生效。

但是这种简单的机制也不是没有缺点。`person`上的属性是被所有的`artist`共享的。在现实世界中，每个`artist`有一些独特的不能和他人共享的属性，比如，他的姓名、流派、代表作等等，于是我的代码可能要改成这样：

```js
//公共属性写在原型上
var person = {
    foo: bar
}

//创建对象
var artist = Object.create(person)

//自己的属性
artist.name = 'Leonardo da Vinci'
artist.school = 'classical'
artist.works = ['Mona Lisa']

```

但是，每次创建一个`artist`对象都要写这么一大串，我们最好把这个创建的过程用一个函数封装起来，即：

```js
var person = {}

function genArtist(name, school, work){
    //设定原型
    var artist = Object.create(person)

    //自有的属性    
    artist.name = name
    artist.school = school
    article.work = work
    
    return artist
}

var vinci = genArtist('Leonardo da Vinci', 'classical', 'Mona Lisa')

//classical
console.log(vinci.school)
```

现在每次通过`genArtist`函数创建新的`artist`时，返回的对象既会继承`person`，又会生成它自己的独有属性。而且，你还可以在创建`person`本身时，把它的`[[prototype]]`指向另一个更基础的对象，从而延长`artist`的原型链。

至此，我们已经创建了一个相当令人满意的纯`Javascript`式的继承机制。

现在我们可以回过头来看看上面讲到的神秘的构造函数和`new`机制。实际上，它的原理和`genArtist`函数的原理是一样的。不同的仅仅是：

1. 你需要生`new`关键字调用函数。

2. 函数内部末尾不需要显式返回对象，因为`new`关键字调用会自动帮你返回。

3. 你不需要通过`Object.create(...)`来指定生成的对象的`[[prototype]]`指针，因为它会自动指向函数本身的`prototype`属性。

__需要注意的是__，函数的`prototype`属性并不是上面的`[[prototype]]`指针，它只是每个*函数对象*都自带的一个属性，其唯一作用就是当用`new`调用时，提供给生成的新对象作为新生成对象的`[[prototype]]`指针的目标（我不知道当初制定标准时把它命名为`prototype`是什么心态，这样很容易造成误解）。

构造函数：

```js
function Artist(name, school, work){
    this.name = name
    this.school = school
    this.work = work
}

var artist = new Artist()

console.log(artist instanceof Artist) //true
```

实际上完全等价于：

```js
function Artist(name, school, work){
    var artist = Object.create(Artist.prototype)
    
    artist.name = name
    artist.school = school
    artist.work = work
    
    return artist
}

var artist = Artist()
console.log(artist instanceof Artist) //true
```

图示如下：

![构造函数和new机制图示](http://placeholder.exp)