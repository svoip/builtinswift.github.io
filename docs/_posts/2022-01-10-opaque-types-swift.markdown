---
layout: post
title:  "Opaque types in Swift"
date:   2022-01-10 20:38:56 +0200
categories: articles

---
Opaque types

Added to Swift in 2019 this feature is somehow mainstream now. 
It has been added to Swift language as a way to "nest generic types". These 3 words open a whole lot of new terms and tradeoffs among them.

CONTINUE:
https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md


Opaque types offer newer ways to design APIs. Similar to generics, opaque types work in close collaboration with protocols.

# protocol
Protocols -> Generics
      -> Opaque types


Protocol is an abstraction and allows more flexibility in what they store. A simple protocol with 1 property is happy once the concrete type implements that property.


# generics
Generics allows to add behavior which can be applied to a set of concrete types (hidden by a same protocol).

-constrains
-callers have to define which particular concrete type is being

Once I have my concrete instance, to check whether it conforms to a protocol, all I do is as? check:
ie, 
if let myType = myObject as? MyProtocol {

} 
OR? (check this)
if myObject is MyProtocol {

}

Once my extend my protocol with a concrete type, "returning to it" is not given, and one has to do the check as above for that. (looser API contract)


# opaque types
It is similar to generics in that it also hides the actual type. But there is one thing it keep strict: the concrete type has to remain the same concrete type, or "preserve type identity".
So one might say, opaque types are "abstract" and "concrete" at the same time.

A function that returns opaque type returns always a single (concrete?) type.
ie,
func makeSomething() -> some MyProtocol {}
implmentor (ie, module) knows about what concrete type is being used and returned to the caller, WHILE not exposing it to outside (ie, app)
where
the app can continue treating the return type as any "myProtocol", while module preserving the right to modify the concrete type being used as it needs to.


# Generics vs. Opaque
why apply says it is the reverse of generics???

generics
-caller defines the concrete type being requested

opaque
-callee defines the concrete type being used



# why is opaque type useful?
- Opaque types allow "nesting"

> opaque has only to do with functions, but not others??
Yes, it is made for "function return types".

How do you distinguish it in code?
With "some" keyword.

# Opaque vs. Protocols
protocols
-any implementor of the protocol can be returned
-wide possibility of types to return
-NESTING: "Protocol X cannot conform to itself"

opaque
-single type must be returned from the "fx return points"
-while preserving the identify of the type
-NESTING YES

..
# from apple site:

Hiding type information is useful at boundaries between a module and code that calls into the module, because the underlying type of the return value can remain private. Unlike returning a value whose type is a protocol type, opaque types preserve type identity—the compiler has access to the type information, but clients of the module don’t.
ie,
module:
-protocol
-concrete type1
-concrete type2

app:
...





--------------- old ----------------

Even though we have been writing code exclusively with ARC for a while now, Cocoa frameworks were created before ARC itself came into existence.

But Apple has been updating these frameworks in a rigorous way. Below we will some examples of such changes. But, let's see the basics first.

## ARC basics

ARC is a compiler feature that removes the burden of memory management from developers. With ARC, we don’t have to (actually we cannot) `retain` or `release` objects, but we can define our objects’ lifetime using attributes like `strong` or `weak`.

With a `strong`ly declared property attribute, we imply that “we are strongly interested in keeping this object in the heap”. Such declaration retains the new object on assignment:


{% highlight objc %}
// inside a custom class
@interface MyCustomClass()
@property (strong) Foo *foo;
@end

@implementation MyCustomClass
Foo *foo = [Foo new];
self.foo = foo;
//self is strongly interested in 'foo' object
//the retain count of 'foo' object now is 1

anotherCustomObj.foo = foo;
//anotherCustomObj is strongly interested in 'foo' object as well
//the retain count of 'foo' object now is 2
@end
{% endhighlight %}


With a `weak` declaration we imply that “we are interested in this object as long as someone else is keeping it alive for us”. Such declaration will not retain the new object on assignment, yet it sets the pointer to `nil` when the referred object is deallocated.

{% highlight objc %}
// inside a custom class
@interface MyCustomClass()
@property (weak) Foo *foo;
@end

@implementation MyCustomClass
Foo *foo = [Foo new];
self.foo = foo; 
// self won't care if 'foo' instance isn't kept around 
// and won't throw error on such occasions 
@end
{% endhighlight %}

Both attributes have been introduced as part of ARC in the past years. Their roughly equivalents in non-ARC code are `retain` and `assign` (although `assign` doesn’t set pointer to `nil` when deallocated). `Retain` and `assign` declarations exist in Cocoa classes even today. We have to pay attention how such classes are implemented in order to avoid memory related pitfalls.



## Recent SDK changes

#### UITableView

Consider the following example. 
Let’s say that we are running it on an iOS 8.0 environment. We set the data source for our table view without keeping a reference to it:

{% highlight objc %}
// inside a custom table view controller
@interface CustomTableViewController()
@property (strong) DataSource *dataSource;
@end

@implementation CustomTableViewController
-(void)viewDidLoad
{
  DataSource *ds = [[DataSource alloc] init];
  self.tableView.dataSource = ds;
  // ...
  // missing line!
  // self.dataSource = ds;
}
@end
{% endhighlight %}


By the time this method returns, `ds` will be deallocated. When the table view starts querying its `.dataSource` property, app will crash with `EXC_BAD_ACCESS`.


![image-title-here](/images/exc_bad_access.png){:class="img-responsive"}



When we edit the scheme of our app and enable NSZombies under Diagnostics, we can clearly see what caused the crash. It is a “message sent to a deallocated object” exception.

![image-title-here](/images/datasource.png){:class="img-responsive"}



Running the same code on an iOS 9.0 environment has a slightly different consequence: it won’t crash. The table view won’t display anything since its data source has been deallocated, but the good thing is that it won’t crash either. 

Why? 

Because the way table view keeps reference to its data source has been changed in UIKit. Up until iOS 8.0 it used to be:

{% highlight objc %}
@property (nonatomic, assign) id<UITableViewDataSource> dataSource
{% endhighlight %}

but since iOS 9.0 it is:

{% highlight objc %}
@property (nonatomic, weak, nullable) id<UITableViewDataSource> dataSource
{% endhighlight %}

The new UIKit behavior is more “tolerant”: if `dataSource` is still there when the table view is loaded up, the table view will call query it in order to populate cells; otherwise no effect.

Let’s see what these definitions "translate" into in real time:

iOS 8.0:
{% highlight objc %}
// inside a custom table view controller
@interface CustomTableViewController()
@property (strong) DataSource *dataSource;
@end

@implementation CustomTableViewController
- (void)viewDidLoad
{
  DataSource *ds = [[DataSource alloc] init];
  // the tableView's .dataSource property is assigned
  self.tableView.dataSource = ds;
 
  //...
  // missing line!
  // self.dataSource = ds;
 
  // the 'ds' data source object will be deallocated now, 
  // WHILE the tableView's .dataSource remains assigned 
}
@end
{% endhighlight %}

iOS 9.0:
{% highlight objc %}
// inside a custom table view controller
@interface CustomTableViewController()
@property (strong) DataSource *dataSource;
@end

@implementation CustomTableViewController
-(void)viewDidLoad
{
  DataSource *ds = [[DataSource alloc] init];
  // the tableView's .dataSource property is assigned
  self.tableView.dataSource = ds;
   
  // ...
  // missing line!
  // self.dataSource = ds;
   
  // 'ds' data source object will be deallocated now,
  // and the tableView's .dataSource will be set to nil
}
@end
{% endhighlight %}


Following this <a href="https://developer.apple.com/library/content/releasenotes/General/iOS90APIDiffs/Objective-C/UIKit.html"  target="_blank">link</a>, you can find UIKit’s several other changes in iOS 9.0, like:
{% highlight objc %}
UITableViewDelegate
UIAccelerometerDelegate
UICollectionViewDataSource
UICollisionBehaviorDelegate
UIDocumentInteractionControllerDelegate
UIGestureRecognizerDelegate
UINavigationControllerDelegate
UIImagePickerControllerDelegate
UIPickerViewDataSource
UIPickerViewDelegate
UIPopoverControllerDelegate
{% endhighlight %}


### NSNotificationCenter


In pre-iOS 9.0 environments, notification center used to use `assign` attribute in relation to its observers. If any of the observers has been deallocated while notification is emitted, then crash was guaranteed. Thanks to the latest SDK changes, we don't have to keep strong references to observers anymore.


