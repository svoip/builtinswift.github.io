---
layout: post
title:  "UIRefreshControl: iOS 10 update"
date:   2016-06-28 20:38:56 +0200
categories: articles

---


Refresh control (`UIRefreshControl`) was introduced in iOS6, and it was meant to be used with table view controllers (`UITableViewController`). 

Table view controller manages the `position` and the `size` of the refresh control while we have to configure manually the `target` and `action`. A use case would look like this:

This is what I had implemented back then in my table view controller.

{% highlight objc %}
UIRefreshControl *refreshControl = [[UIRefreshControl alloc] init];
[refreshControl addTarget:self action:@selector(refreshing:)
 forControlEvents:UIControlEventValueChanged];
[self.tableView addSubview:refreshControl];
 
-(void)refreshing:(UIRefreshControl *)aRefreshControl
{
 // start off asynchronous work, like, network calls
 //..
 // close the refresh control when done
 // ! assure that it is the main thread
 [aRefreshControl endRefreshing];
}
{% endhighlight %}


iOS 10.0 introduces a new protocol, `UIRefreshControlHosting`, which is a common interface to use refresh control. The protocol has only one property; scroll view and table view controller adopt this protocol, as such, these 2 classes support refresh control “out of the box”. Same as before, the `position` and the `size` of the control is handled by the system.

{% highlight objc %}
UIRefreshControlHosting
//the only property to adopt
.refreshControl
{% endhighlight %}

An implementation in Swift looks like this:
{% highlight swift %}
let refreshControl = UIRefreshControl.init()
let attributes = [NSForegroundColorAttributeName: UIColor.black()]
let attributedString = AttributedString(string: "Loading...", attributes: attributes)
refreshControl.attributedTitle = attributedString
refreshControl.tintColor = UIColor.red()
refreshControl.addTarget(self, action: #selector(refreshing), for: .valueChanged)
// attach it to scrollview
scrollView.refreshControl = refreshControl
 
func refreshing(){
  // asynchronous work, like network calls
  //..
  // close the refresh control when done
  self.scrollView.refreshControl?.endRefreshing()
}
{% endhighlight %}

![image-title-here](/images/refresh-control.png){:class="img-responsive"}

