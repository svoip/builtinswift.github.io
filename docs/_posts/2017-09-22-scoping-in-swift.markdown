---
layout: post
title:  "Scoping and its uses in Swift"
date:   2017-09-22 00:00:00 +0200
categories: articles

---

If `module`s contain the programming constructs we deal with every day, `namespace` references them for easy access (and to avoid `Use of undeclared type` errors), and `scope` enforces the order of referencing. 

So in a nutshell, `module` is a container, `namespace` is a referencer, `scope` is an enforcer. That was easy, right?
Now in wider terms. 

### Namespace, module and scope


An Xcode project has an implicit namespace, and it keeps constructs (in Apple terms, `member`s) we write in the form of a hashmap.
A construct can be anything from class, class property, method to structs and enums.

{% highlight swift %}
// hypothetically
[member1: itsAddress, member2: itsAddress]
// or in practice
[MyClass:itsAddress, myObj:itsAddress, varIjustCreated:itsAddress, ...]

{% endhighlight %}

So whenever we write
{% highlight swift %}
let myObj: MyClass
{% endhighlight %}

compiler knows what we are talking about.

When we import a module into the project, this implicit namespace grows. 

{% highlight swift %}
// suppose we have imported ModuleX
[MyClass:itsAddress, myObj:itsAddress, varIjustCreated:itsAddress, 
ModuleX.ItsClass1: itsAddress, ModuleX.itsSomeGlobalConstant: itsAddress, ...]

{% endhighlight %}


Where does the scope stand? 

At any given context in our code, scope looks through the namespace and applies the innermost local namespace available. It works from bottom up as in this graph below.
![alert-controller](/images/scope-arrow.png){:class="img-responsive"}


A concise example:
{% highlight swift %}
let myValue = 0
class MyClass {
  let myValue = 5 
  init(){
   print("myValue is: \(myValue)")
  }
}

let myObj = MyClass()
// myValue is: 5

{% endhighlight %}

![alert-controller](/images/scope-colored.png){:class="img-responsive"}




How does Swift employ scope? 


### Scoping in Swift


#### 1. Optional binding

One of the principal features of Swift, optional binding, uses scope.
{% highlight swift %}
var foo:String?
// 'foo' is an Optional String
if let foo = foo {
  // 'foo' is a String
}
// 'foo' is an Optional String again
{% endhighlight %}

or similar example:

![alert-controller](/images/animal-scratch.png){:class="img-responsive"}

With `guard let` scope can be changed to assist readability:

{% highlight swift %}

let password:UITextField
// 'password' is a UI control
guard let password = password.text else { return }
// now 'password' is its text value
{% endhighlight %}


#### 2. Chaining

Something that Objective-C never allowed, Swift permits nested scopes, giving us an extra tool against naming collisions or for a better code organization.

{% highlight swift %}
struct Foo {
  struct Bar {
    struct Baz {
    	// 
    }
  }
}

let baz = Foo.Bar.Baz()

{% endhighlight %}

I like to use this technique as a configuration setting in my projects. While typing, Xcode's code completion brings even more delight.
{% highlight swift %}

// use the recent token or device-id 
let token = MyApp.Defaults.Token.Get()
let deviceId = MyApp.Defaults.DeviceId.Get()

// cache the next value
let newToken = "my next token"
MyApp.Defaults.Token.Set(newToken)

// fetch the correct color
myLabel.textColor = MyApp.Color.textColor

{% endhighlight %}


{% highlight swift %}
// which is implemented as:
struct MyApp {
  struct Defaults {

    struct Token {
      static func Get()->String?{
        return ...
      }
      static func Set(_ val:String?){
        if let val = val {
          // save the new token in, for example, NSUserDefaults
        }
        else {
         // or remove the known token
        }
      }
    }

    struct DeviceId {
    	//
    }
  }
  
  struct Color {
    static let titleColor = UIColor.white
    static let textColor = UIColor.gray
    // ...
  }
}
{% endhighlight %}


(seems gone) give Firebase's API as example


#### 3. Strong-typing mechanism

Many objects fetched from remote endpoints contain 'id' property, which typically is the unique identifier of that object. It can be `Int` or `String` depending on the circumstances. If it is an `Int`, can 'id's of two people be added? What would that even mean?

{% highlight swift %}
struct Person {
  var id:Int
}
{% endhighlight %}

We can implement this as: 
{% highlight swift %}

struct Person {
  struct ID {
    var value:Int
  }
  var id:ID
}
{% endhighlight %}

and we get a strongly-typed member, declared in the scope of the class itself.

![alert-controller](/images/person-id.png){:class="img-responsive"}



