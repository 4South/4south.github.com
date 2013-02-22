---
layout: post
title: "Layering Canvases with Ember.ContainerView"
date: 2013-02-22 01:30
comments: true
categories: View 
published: true 
---
##Dependency Versions
This post was built with Ember.js 1.0.0RC1, handlebars RC3, and jQuery 1.9.1
##Understand The Goal
###What will we explore?
Our goal is to use Ember's **ContainerView** class to wrap up **multiple HTML5 canvas elements** into a single, layered display.  This pattern is extremely
common for effective use of canvas elements and Ember offers an elegant solution for encapsulating them as a "unit".
###Show me the end product before I listen...
Feel free to check out this <a href="https://jsfiddle.net/skane/msvDW/34/">Completed Fiddle</a> or build your own fiddle as we go.
##Separating Signals and Noise
This project, and many that follow require some "support" code to achieve the end goal.  Often times that code is not germane to the point the article is trying
to make about Ember.js.
###Signals
Use of Container/Canvas Views<br />
Dynamic styling using computed properties<br />
Initial Canvas drawing and re-rendering<br />
###Noise
Buttons to change linecount and supporting behavior validation<br />
Local variable setup (an unfortunate necessity in Ember..)<br />
Methods for drawing to canvas (these are worth exploring but don't directly convey our purpose here)<br />
##Ember setup
```coffeescript
window.App = Em.Application.create()

App.ApplicationController = Em.Controller.extend
  #we use this instead of the more typical "content" to avoid making Application Controller angry
  appVars: Em.Object.create
    height: 300
    width: 300
    lineCount: 3

App.Canvas = Em.View.extend
  tagName: "canvas"
  isVertical: true 
  didInsertElement: ()->
    @_super()
    @fill()
  
  fill: ()->
    #this will be our primary drawing method (to be continued...)
    return

App.CompositeView = Em.ContainerView.extend
  tagName: "div"
  childViews: ['canvas1', 'canvas2']
  
  canvas1: App.Canvas.create
    isVertical: false
    color1: "blue"
    color2: "grey"
  canvas2: App.Canvas.create
    color1: "red"
    color2: "white"
```
###What is all this...stuff?
The **ApplicationController** is the context for all our views in this system.  It will eventually also house a few basic methods to support some html buttons.<br />
The **CompositeView** is an instance of Ember's ContainerView and is used to hold the two canvases (which contain the bulk of the program's code)<br />
The **Canvas Views** are setup to wrap HTML5 canvas elements (see previous blogpost for details).  We instantiate two of them as we intend to build a layered display.
##Add a template and give our views some style
{% codeblock lang:html %}
<script type="text/x-handlebars>
  <p>Test to confirm the template is loading</p>
  {{ "{{ view 'App.CompositeView' contentBinding='appVars' " }}}}
</script>
{% endcodeblock %}
This template is our **application template** (Ember automatically assigns this if no **data-template-name** is declared in the script tag).  We add a line to confirm 
our app is rendering this template and we add a handlebars tag to create an instance of our view and bind its content to the applicationController's **appVars** attribute.
```coffeescript
App.CompositeView = Ember.ContainerView.extend
    tagName: "div"
    #this computed property will create a CSS style string 
    style: (->
        "height:" + @get('content.height') + "px;" + "width:" + @get('content.width') + "px;"
    ).property('content.height, content.width')
    attributeBindings: ['style']
    childViews: ['canvas1', 'canvas2']
    
    canvas1: App.Canvas.create
        isVertical:  false
        color1: "blue"
        color2: "grey"
    canvas2: App.Canvas.create
        color1: "red"
        color2: "white"
```
This View is now complete and will not change for the rest of this post.  We have added a **computed property called "style"** to the view which constructs a string of 
in-line styles to be added onto the view's element, "div", via the **attributeBindings** attribute.  Read <a href="http://emberjs.com/api/classes/Ember.View.html">
Ember's View API</a> for more information on how these attributes work.<br />
**NOTE:** This is not the only way to style an element but it showcases a method that will allow us to **dynamically re-size our view** if the view's content.height or content.width are changed by our application.
```coffeescript
App.Canvas = Ember.View.extend
    tagName: "canvas"
    contentBinding: "parentView.content"
    attributeBindings: ['height', 'width']
    height: (->
        @get "content.height"
    ).property('content.height')
    width: (->
        @get "content.width"
    ).property('content.width')
    isVertical: true
    
    layoutChanged: (->
        @fill()
    ).observes('content.lineCount', 'content.height', 'content.width')
    
    didInsertElement: ()->
        @_super()
        @fill()
    
    fill: () ->
        return
```
We have again added an attributeBindings method to our view but, critically, we have utilized it differently.  In the CompositeView we used **in-line style** to set 
our view's height and width.  Here, we must use the **html attributes "height" and "width"** to give dimensions to a canvas element.  This is an important 
distinction.<br />
Attributes **height** and **width** are implemented as computed properties that simply reflect **content.height** and **content.width**.  This again allows us 
to **re-size our canvases** elsewhere in our application should we want to do so.
```css
div {
  position: relative;
}
canvas {
  opacity: .5;
  position: absolute;
}
```
Setting position:absolute on the canvas means they will draw directly on top of our div element rather than in normal html block format.  The results of this aren't yet apparent but they will be shortly.  We set opacity so that our layers are partially transparent.
##Canvas drawing code
```coffeescript
App.Canvas = Ember.View.extend
    tagName: "canvas"
    contentBinding: "parentView.content"
    attributeBindings: ['height', 'width']
    height: (->
        @get "content.height"
    ).property('content.height')
    width: (->
        @get "content.width"
    ).property('content.width')
    isVertical: true
    
    layoutChanged: (->
        @fill()
    ).observes('content.lineCount', 'content.height', 'content.width')
    
    didInsertElement: ()->
        @_super()
        @fill()
    
    fill: () ->
        isVertical = @get "isVertical"
        lineCount = @get "content.lineCount"
        el = @get "element"
        height = @get "content.height"
        width = @get "content.width"
        color1 = @get "color1"
        color2 =@get "color2"
        if el
            ctx = el.getContext "2d"
            for lineNum in [1..lineCount]
                color = if lineNum%2 is 0 then color2 else color1
                @drawRect ctx, color, lineNum, lineCount, isVertical, height, width
    #helper method that draws each uniquely-colored box
    drawRect: (ctx, color, lineNum, lineCount, isVertical, height, width) ->
        if isVertical
            rHeight = height
            rStartY = 0
            rWidth = width/lineCount
            rStartX = width/lineCount * (lineNum-1)
         else 
            rHeight = height/lineCount
            rStartY = height/lineCount * (lineNum-1)
            rWidth = width
            rStartX = 0
         ctx.fillStyle = color
         ctx.fillRect rStartX, rStartY, rWidth, rHeight
```
This class is now finished and will not change for the rest of this post.<br />
This code looks a little dense but its purpose is very simple.  It draws **horizontal** or **vertical** stripes onto our canvases using the low-level canvas API's methods.  It is easy to google these methods so I will not explain them here.  The rest of the lines are dedicated to calculating x,y,height, and width based on our view's content (which is inherited from applicationController.appVars).  **Feel free to tweet, email, or comment below if any of this is unclear**.  <br />
**NOTE: ** Be sure to remove the "return" we had listed in the fill method initially.  It was only there as filler.<br />
Finally, we also implement an Ember observer called **layoutChanged which fires any time content.lineCount, content.height, or content.width change**.  We use this to signal to our canvas that it must re-draw itself.  We don't need to clear the canvas in this particular app because our draw process completely re-draws the whole canvas.  **This may not always be the case!**
##Dynamically re-draw our canvas by changing lineCount!
```coffeescript
App.ApplicationController = Ember.Controller.extend
    appVars: Ember.Object.create
        height: 300
        width: 300
        lineCount: 3
     #methods to change lineCount within range 1->10
     upLineCount: () ->
        if @get("appVars.lineCount") <=9
            @incrementProperty "appVars.lineCount"
        else
            @set "appVars.lineCount", 10
     downLineCount: () ->
        if @get("appVars.lineCount") >=1
            @decrementProperty "appVars.lineCount"
        else
            @set "appVars.lineCount", 0
```
These new methods on the applicationController change the lineCount attribute on appVars within the range 1->10.  We will call these methods from our template as shown below.
{% codeblock lang:html %}
<script type="text/x-handlebars">
    <p>Use buttons to change line density</p>
    <button {{ "{{action 'upLineCount' " }}}}>Increase</button>
    <button {{ "{{action 'downLineCount' " }}}}>Decrease</button>
    {{ "{{ controller.appVars.lineCount " }}}}
    {{ "{{ view 'App.CompositeView' contentBinding='appVars' " }}}}
</script>
{% endcodeblock %}
We have added buttons that utilize Ember's **action** handlebars tag to call the new methods on applicationController.  We now have a way for users to change the lineCount which creates such dreamy, dreamy patterns...
##Conclusion and future work
Once again, here is a <a href="http://jsfiddle.net/skane/msvDW/34/">Completed Fiddle</a> showcasing this application. 
This post has highlighted a pattern that we intend to elaborate on with examples of **UI widgets, editable objects, and more advanced canvas APIs**.  If you understand what is going on in this post and in the previous canvas post you will be prepared to do some truly useful things in Ember.js!  Who doesn't love being useful?  Cats...that's who.<br />
**/golfclap<br />
/bow**
