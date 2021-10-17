---
layout: post
title:  "Handling HTTP responses with NSURLSession API"
date:   2016-05-08 20:38:56 +0200
categories: articles
---


When we use NSURLSession API, how our apps will be notified about the incoming HTTP responses largely depend on:

- how we configure “url session”, and,
- how we configure each single “url session data task”


### NSURLSession
First, we need to create a url session object. We pass in parameters such as delegate (the entry point for handling most of, if not all, incoming responses) and queue (the operation queue on which the responses will be presented to our app).

Why would we want to specify the queue? Shouldn’t we always use the main queue?

Suppose, we want to do some processing on the incoming data before presenting it to the user (for example, querying database and figuring out if we have already consumed such data), so we should opt for a background queue, which is also the default setting.

For example:

{% highlight objc %}
NSURLSessionConfiguration *config;
NSOperationQueue *queue;
NSURLSession *session;
 
config = [NSURLSessionConfiguration defaultSessionConfiguration];
 
// Possible 'queue' options:
queue = [NSOperationQueue mainQueue];
//queue = [NSOperationQueue new];
//queue = nil; it is equivalent to the above line
 
session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:queue];
 
// In this example, we are expecting the responses on the main queue
// and the delegate is 'self'


{% endhighlight %}

, or a rather simple setting:

{% highlight objc %}

NSURLSession *session;
session = [NSURLSession sharedSession];

// Here both ‘queue’ and ‘delegate’ are nil,
// ..so it is assumed, in such cases, that down the road we use
// ..completion handler APIs (discussed below)

{% endhighlight %}



### Data task
Then we need to configure a single request when we initialize a data task. 

For example:

{% highlight objc %}

NSURLSessionUploadTask *uploadTask;
NSURL *fileURL = ...;
NSMutableRequest *request = ...;
uploadTask = [session uploadTaskWithRequest:request fromFile:fileURL];
[uploadTask resume];
// for this upload task 'delegate' methods will be called
//..(if we have set any delegate object and implemented the
//..delegate methods)
 
// OR
 
typedef void (^RequestCompletionHandler)(NSData *data, NSURLResponse *response, NSError *error);
RequestCompletionHandler completionHandler = ...;
uploadTask = [session uploadTaskWithRequest:request fromFile:fileURL completionHandler:completionHandler];
[uploadTask resume];
// for this upload task, the 'completion handler' will be called
{% endhighlight %}


Question arises: 

- What happens if we don’t set any delegate, and we don’t use a completion handler for a task?

System won’t inform us about the networking result. We shouldn’t be doing this, as it would make no sense.


- What happens if we specify both the delegate and a completion handler?

This is rather interesting. 

We will be informed by both delegate methods and the completion handler, though completion handlers can do only a portion of what delegate methods can do for us. In such cases, completion handlers substitute this sole method of `NSURLSessionTaskDelegate` protocol (Note that this is the “super” protocol for all the task types):

{% highlight objc %}
// Tells the delegate that the task finished transferring data.
URLSession:task:didCompleteWithError:
{% endhighlight %}


So, possible set of steps (for uploading task) may look like this:

{% highlight objc %}
// data task is prepared and request started
a) URLSession: task: didSendBodyData: totalBytesSent: totalBytesExpectedToSend:
(called multiple times as necessary)
b) URLSession: dataTask: didReceiveData: 
c) URLSession: task: didCompleteWithError: 
...
{% endhighlight %}

or,

{% highlight objc %}
// data task is prepared and request started
a) URLSession: task: didSendBodyData: totalBytesSent: totalBytesExpectedToSend: 
(called multiple times as necessary)
b) URLSession: dataTask: didReceiveData: 
c) the completion handler invoked
...
{% endhighlight %}

But the real power of using a delegate is when there are extra layers (such as authentication, caching, progress handling, etc) and when we have to use a background session (which wouldn’t be possible without the use of delegates).


### Additional notes
#### Completion handler and background session

Calling “completion handler based” APIs with background sessions results in runtime error:

> Terminating app due to uncaught exception NSGenericException, 
reason: Completion handler blocks are not supported in background sessions. 
Use a delegate instead.

It seems that it is just an NSAssert. It would be better if it were a compile time error rather than a runtime error, thus notifying developers early on.

#### Client-side errors
Errors returned through delegate methods or completion handlers refer to client-side only.

Recently I was trying to upload an image using uploadTaskWithRequest:fromFile:, and the url of the file to upload was resolving to `nil`. 
The system would then cancel the request right away with a log:

{% highlight objc %}
Error Domain=NSURLErrorDomain Code=-999 "cancelled" 
UserInfo={NSErrorFailingURLKey=http://<my-url>, 
NSErrorFailingURLStringKey=http://<my-url>, 
NSLocalizedDescription=cancelled
{% endhighlight %}

It would have been better if the log was more informative, like “the request is cancelled because there was no file to upload”. It just means that we have to add more checks in our code, rather than relying on errors or logs from the system.


#### Server-side errors

To learn existance of a server-side error, we need to downcast NSURLResponse object to NSHTTPURLResponse and query its `statusCode`.
