---
layout: post
title:  "API updates in iOS SDK with ARC"
date:   2016-01-16 20:38:56 +0200
categories: articles

---

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


