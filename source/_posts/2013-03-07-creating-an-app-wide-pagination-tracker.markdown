---
layout: post
title: "Creating an App-Wide Pagination Tracker"
date: 2013-03-07 01:44
comments: true
categories: Application
published: true
---
##Dependency versions 
This post was built with Ember.js 1.0.0RC1, handlebars RC3, and jQuery 1.9.1
##Understand the goal 
###What is the point?
Recently, I was discussing a mechanism for storing json metadata in an Ember application and that thought experiment led me to this possible solution.  We will build a simple jSfiddle that demonstrates a possible solution to this problem that utilizes several interesting Ember-isms including **injections, simulated Ajax with the Runloop, and the didInsertElement hook on Ember.Views**.
###Show me this...fiddle
Feel free to check out this <a href="http://jsfiddle.net/skane/sLubH/12/">Completed Fiddle</a> or build your own fiddle as we go.
##Ember application setup and a custom initializer!
```coffeescript
window.App = Em.Application.create()

#we want this object made as a singleton injected on controllers/routes/store
App.PaginationTracker = Ember.Object.extend
    shitHasChanged: false
    model1pagination: 5
    model2pagination: 10

#read about these...they are awesome.  Check the Ember source
Ember.Application.initializer
    name: "pagetracker"
    initialize: (container, application) ->
        #declare that we want a single instance of pagetracker app-wide
        container.optionsForType('pagetracker', {singleton: true})
        #register our pagetracker singleton with the container 
        container.register('pagetracker', 'main', application.PaginationTracker)
        #inject pagetracker onto all controllers/routes and the store
        container.typeInjection('controller', 'pagetracker', 'pagetracker:main')
        container.typeInjection('route', 'pagetracker', 'pagetracker:main')
        container.injection('store:main', 'pagetracker', 'pagetracker:main')
```
###What the ... is all this mess?
Don't worry this is easy!  It just looks messy because you are a hopeless 
minimalist obsessing over LOC control.  Remember the goal was to **inject** 
a single instance of our new **paginationtracker** object onto all controllers, 
routes, and the store so that we can use the data to affect our displays.<br />
First, we create an app and define our PaginationTracker object by extending Ember.Object.<br />
Second, we create a custom **Ember.Application.initializer** to handle our 
injection.  Read more about this awesome functionality at 
<a href="https://github.com/emberjs/ember.js/blob/v1.0.0-rc.1/packages/ember-application/lib/system/application.js#L100">Ember Application Source</a><br>
We now have an Ember application with the setup desired...sadly we can see
no proof that my claims are valid.  Let's fix that!
##Fake ajax with Ember.run to change paginationtracker
{% codeblock lang:html %}
<script type="text/x-handlebars">
    <h4>model1pagination is : {{ "{{pagetracker.model1pagination" }}}}</h4>
    <h4>model2pagination is : {{ "{{pagetracker.model2pagination" }}}}</h4>
    {{ "{{#if pagetracker.shitHasChanged" }}}}
        <h2>Pagination Changed!  Infront of your eyes!</h2>
    {{ "{{/if" }}}}
</script>
{% endcodeblock %}
###Quick explanation
We are using the default template (not named explicitly because Ember will
automagically name it "application" for us).  We want to display model1pagination 
and model2pagination from an attribute called "pagetracker" (our pagination 
tracker singleton that we just **injected onto all controllers**) on our 
application controller.<br />  
Finally, we show a little message if our data has changed (just for
fun, though this value is ALSO stored on the paginationtracker).
```coffeescript
App.ApplicationController = Ember.Controller.extend
    #fake ajax method
    getMyTotallyFakeAjaxData: () ->
        #here we fire our fake ajax call using Ember.run.later (a sort of
        #replacement for setTimeout
        Ember.run.later(@, @updatePageTracker, 4000)
        
    #fake ajax "callback"
    updatePageTracker: () ->
        #set properties on our pagetracker singleton through this controller
        @get('pagetracker').set('model1pagination', 10)
        @get('pagetracker').set('model2pagination', 30)
        @get('pagetracker').set('shitHasChanged', true)
        

App.ApplicationView = Ember.View.extend
    #engage our controller's fake ajax call when this view is inserted in the DOM
    didInsertElement: () ->
        @_super()
        @get('controller').getMyTotallyFakeAjaxData()
```
###This is cool...what is it?
We want our view to fire off a "fake ajax-y method" on our controller when it 
is first inserted into the DOM.  The initial pagination values are displayed 
for 4 seconds before they are replaced by updated values.
###Go home point-of-this-article...you're drunk
The key thing to understand from this code is that we are accessing a singleton
instance of our **paginationtracker** through our applicationController 
thanks to our clever injection!  <br />
We can update its properties and then use standard handlebars lookup paths to 
access that object to display its current attribute values.  This is wonderful 
as we could now use these values in any controller, route, or even directly
on the store to make decisions about our application's behavior.  <br />
yay!
###Unsolicited opinion
Ember is really not hard once you learn to stop fighting it and instead LEVERAGE
its conventions to achieve world domination...like Zuckerberg..but cooler.
##Upcoming blogs (omg see the future)
I want to explore some more advanced Ember use-cases including **undo/redo, animation, and extracting view attributes from CSS stylesheets (black.fuckin.magic)**.
As always, follow @stv_kn for updates on stuff and links to useful fiddles.
