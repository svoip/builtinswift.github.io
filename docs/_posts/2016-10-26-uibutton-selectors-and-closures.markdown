---
layout: post
title:  "UIButton selectors and closures"
date:   2016-10-26 20:38:56 +0200
categories: articles
---

How can we bind UIButton to an action in a single line code?

Recently I was creating a "compound UI components" architecture with different view levels. I prepare configuration objects with necessary configuration info, and at runtime UIKit elements are constructed on the basis of the passed in parameters. For labels and views that only read and display data there was no hussle.

{% highlight swift %}
let compoundObj = MyCompoundObj(
	text: "Text to display in a label",  
	color: UIColor.red // color background for a view
	 )
array.append(compoundObj)


...
let compoundObj = ... // get compound object at an index
let label = UILabel()
label.text = configObj.text
view.addSubview(label)
...

{% endhighlight %}

But for buttons and their callbacks I encountered the following problem.


{% highlight swift %}

let compound = MyCompoundObj(
	text: "Text to display in a label",  
	color: UIColor.red,
	btnTitle: "Click me!",
	btnCallback: <some closure to bind with a button>
	 )
array.append(compoundObj)

let compoundObj = ... // get compound object at an index
let button = UIButton.init(.custom)
button.setTitle(compoundObj.btnTitle), for.normal
button.addTarget... what ?!!

{% endhighlight %}

I encountered <a href="http://stackoverflow.com/questions/25919472/adding-a-closure-as-target-to-a-uibutton/30518764"  target="_blank">this post</a> on stackoverflow, that describes how to implement this. 



{% highlight swift %}
extension UIButton {
private func actionHandleBlock(action:(() -> Void)? = nil) {
    struct __ {
        static var action :(() -> Void)?
    }
    if action != nil {
        __.action = action
    } else {
        __.action?()
    }
}

@objc private func triggerActionHandleBlock() {
    self.actionHandleBlock()
}

func actionHandle(controlEvents control :UIControlEvents, ForAction action:() -> Void) {
    self.actionHandleBlock(action)
    self.addTarget(self, action: "triggerActionHandleBlock", forControlEvents: control)
}
}

... 
let button = UIButton()
button.actionHandle(controlEvents: UIControlEvents.TouchUpInside, ForAction: <some closure>)

{% endhighlight %}

And it worked really great. 

Until I found out that with multiple instances of buttons this implementation won't work. It is implemented as an extension, and there is really one single property to hold onto the closure. However many buttons are created and assigned a closure each, the last closure would override all and whichever button is clicked one single closure would be called.


I decided to subclass UIButton, thus each button instance would have its own 'inner property' to hold onto the closure.


{% highlight swift %}
typealias ButtonClick = (Void) -> Void

// Multiple instances of button have multiple closures (rather than one with the below implementation)
class ClosureButton: UIButton {
    var action: ButtonClick?
    private func actionHandleBlock(action:ButtonClick? = nil) {
        if action != nil {
            self.action = action
        } else {
            self.action?()
        }
    }
    
    @objc private func triggerActionHandleBlock() {
        self.actionHandleBlock()
    }
    
    func actionHandle(controlEvents control:UIControlEvents, forAction action:@escaping () -> Void) {
        self.actionHandleBlock(action: action)
        self.addTarget(self, action: #selector(self.triggerActionHandleBlock), for: control)
    }
    
}

{% endhighlight %}

Now my problem is solved like so:
{% highlight swift %}

let compound = MyCompoundObj(
	text: "Text to display in a label",  
	color: UIColor.red,
	btnTitle: "Click me!",
	btnCallback: <some closure to bind with a button>
	 )
array.append(compoundObj)

let compoundObj = ... // get compound object at an index
let button = UIButton.init(.custom)
button.setTitle(compoundObj.btnTitle), for.normal
button.actionHandle(controlEvents: .touchUpInside, forAction: compoundObj.btnCallback)

{% endhighlight %}


[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
