---
layout: post
title:  "Managing and consuming pods"
date:   2017-01-05 20:38:56 +0200
categories: articles

---

Cocoapods is not only about consuming public pods. We can create and manage our own pods and share them with different team of developers. 

Cocoapods can even help organize dependencies and architectural design (<a href="http://product.hubspot.com/blog/architecting-a-large-ios-app-with-cocoapods" target="_blank">check out how they did</a>). I was inspired by this blog, and through a research, I collected some suggestions for those, who have struggled to grasp the main picture, can follow along.


## Basics

Cocoapods uses a thin "configuration setting", `specs`, in order to group together a set of pods (or say "dependency libraries"). It groups them by name, tag and source location. A copy of the specs is hosted on a remote server so that it is visible to different developers, while a local copy is kept for offline referencing.



Using a real life analogy, `podfile` is "a shopping list", and `podspec` is "a particular product's ingredients list", and `specs` would be "an aisle inside a supermarket" where particular set of products are found, for example, dairies or fresh fruits.
Sample sentences would then look like "we consume what is in the shopping list", "we refer to product's ingredients list to see if it is indeed what we want", "we check different catalogs to see if a product we want is found there".

Thus `podspec` is the basic unit that every pod interacts with. It is the glue that holds together different aspects of Cocoapods. `Specs` and `podfile` help us define our intention towards pods.


The common specs file should look familiar:
{% highlight txt %}
source 'https://github.com/CocoaPods/Specs.git'
{% endhighlight %}
but we can also use a third party (for example, Artsy):
{% highlight txt %}
source 'https://github.com/artsy/Specs.git'
{% endhighlight %}
or create our own:
{% highlight txt %}
source 'https://github.com.org/<your username>/<your specs repo>.git'
{% endhighlight %}

## How it works 

When you run `pod install`, Cocoapods parses every `specs` entry and goes through all the `podspec`s in order to locate and install their relative pods. In development pods, normally source paths are included in `podfile`, thus one can omit `specs` declaration.

Note that a pod must be listed under one of the included `specs`, otherwise, Cocoapods will fail to (locate and consequently) install that pod.

Note that I am using this legend in the following guide:
{% highlight txt %}

'ABC', 'DEF' are names of imaginary pods
'XYZ' is the name of imaginary specs

{% endhighlight %}


### Development (private) pods

It is a pod which is internally being developed. It may end up in the global Cocoapods `specs`, if, for example, we desire to open-source it.

Structure of podfile and podspec: 

{% highlight txt %}
// Podfile
target 'MyProject' do
  pod 'ABC', :path => '../'
end
{% endhighlight %}

{% highlight txt %}
// Podspec
Pod::Spec.new do |s|
s.name = 'ABC'
s.source_files = 'ABC/Classes/**/*'

// dependency pods, if any
s.dependency 'SnapKit'
...

// other details of the pod
...
end

{% endhighlight %}

![image-title-here](/images/dev-pod.png){:class="img-responsive"}


Development pods are better maintained through `specs`. I discuss these steps further below.


### Public pods

{% highlight txt %}
// Podfile
// include every other specs where the pods are referenced
source 'https://github.com/CocoaPods/Specs.git'

target 'MyProject' do
  pod 'ABC', '>0.1.1'
end
{% endhighlight %}

Podspec is the same as development pod.

![image-title-here](/images/public-pod.png){:class="img-responsive"}


### Create and maintain pod and specs

This is the summary of the steps to follow. Further below are the detailed explanations of each step.

#### Specs repo
- prepare a remote repo (ie, bitbucket, github) for `specs`
- add it as a `specs` repo

#### Pod repo
- prepare a remote repo (ie, bitbucket, github) for a private pod
- create a pod
- add this pod as a child of the `specs` repo

#### On a regular basis (as many times as needed)
- commit, tag and push your pod's changes
- push your pod to the `specs` repo




### Specs repo (detailed)

{% highlight txt %}

// create an empty repo and push it to remote
echo "This is a sample specs repo" >> README.md
git init
git add README.md
git commit -m "first commit"
git remote add origin <remote repo where you want to store the specs>
git push -u origin master

{% endhighlight %}

{% highlight txt %}

// inform Cocoapods that you intend to use this as a podspec repo
// create new 'specs repo': a local copy will be created as well (ie in ~/.cocoapods/repos/...)
pod repo add XYZ <the remote repo of the specs>

// (optional) navigate there and check if all pods are 'correct'
pod repo lint . 

{% endhighlight %}


#### Pod repo

{% highlight txt %}

// create a private pod
pod lib create ABC

// inform Cocoapods that it is a private pod
pod lib lint ABC.podspec --private 

pod install --verbose

{% endhighlight %}

{% highlight txt %}

// introduce the pod to your remote repo
git remote add origin <remote repo where you want to store the pod>

git push -u origin master
(eventual changes must be regularly added to git, and pushed to remote)

{% endhighlight %}


#### On a regular basis

{% highlight txt %}

// tag and push relative info to remote. 
// Note. This must be the same as the one specified in the project's podspec file
git tag 1.0.1 
git push origin --tags

// commit and push source code, if any different

{% endhighlight %}

{% highlight txt %}

// lint and push the private repo to the container
// (optional)
pod spec lint ABC.podspec --verbose --allow-warnings 

pod repo push XYZ ABC.podspec --verbose --allow-warnings


{% endhighlight %}



Additional notes:

- running `pod install` on an already installed pod causes it to be updated or removed

- while developing own pod, in order for local and remote copies to communicate with each other, their versions must match

- setting the dependency in a podspec and a podfile have a slightly different syntax, but they carry the same purpose

- pod dependency between projects: while adding a pod to your project, that pod's dependencies will be installed in the same directory with your project's own dependencies (from the project's point of view these pods have no preference over each other)

- `specs` repo being public means that the pods listed in it are publicly visible (but it doesn't necessarily mean these pods are open-sourced)


- importing Objective-C pod into Swift project does not require bridge-header, as long as you specify 'use_frameworks!' in the podfile



