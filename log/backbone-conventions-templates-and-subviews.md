
```yaml
title: "Backbone conventions: templates and subviews"
date: July 22, 2013
author: Chris Joel
```

[Backbone.js][0] is a good JavaScript [MVW][3] toolkit, but it does not define
many of the strong conventions that application developers have come to expect
when working on large projects. [Backbone is a baseline for building out a
larger MVC architecture][1], and not a standalone framework in its own right, so
starting a new project with Backbone will inevitably require developers to
architect some MVC conventions on their own. I have used Backbone (or parts of
it) for a few projects, including form-based interfaces, data visualizations and
games. This series of posts discusses strategies to approach some Backbone
conventions that I have found to be useful and frequently implemented.

I will assume that the reader has at least an intermediate understanding of the
JavaScript language, [DOM][9] and [jQuery][8], and is at least casually familiar
with basic Backbone and [MVC][4] constructs.

## Some background on templates and subviews

Templates are useful because they allow us to break complex and shifting DOM
hierarchies into discrete, reusable chunks. One of the first conventions that is
established in a new Backbone project is how templates should be rendered. This
is due, in no small part, to the fact that [the default 'render' mechanism in
Backbone is implemented as a no-op][2]. Typically, the base view of an extended
Backbone framework will implement template rendering in an overridden render
method.

Another commonly encountered pattern that is not explicitly defined by Backbone
is the concept of views that contain child views, or 'subviews.' This is how
views are described in Backbone's source code:

> Backbone Views are almost more convention than they are actual code. A View
> is simply a JavaScript object that represents a logical chunk of UI in the
> DOM.

If a Backbone ``View`` represents a chunk of UI in the DOM, and the DOM
hierarchy is a tree of such chunks, it follows that your views will be organized
similarly. For example, it is common for a list component view to contain child
list item views. Child views may in turn have child views of their own. These
hierarchies are malleable, so it is useful to establish a conventional mechanism
for managing the relationship between views and their subviews (their children).

Template rendering builds out our DOM hierarchy, and subviews give us a pattern
against which we can organize and manipulate that hierarchy, so I like to
consider rendering templates and organizing subviews as two conventions that are
closely linked to each other.

### View considerations

Backbone Views will always create a root element when instantiated, even when
``render`` is not implemented. By default, this element is a ``div``, but the
default can be overridden by setting the class property ``tagName`` to something
else (e.g., ``'section'``).

The root element is created to support another Backbone ``View`` feature: [event
delegation][5]. In order for ``View`` events to be delegated, the root element
has to exist. It is critical to keep event delegation in mind while dealing with
Backbone rendering features.

## Let's render some templates

The render method for the built-in Backbone ``View``, at the time of this
writing, is implemented something like this:

```javascript
Backbone.View.prototype.render = function() {
  return this;
};
```

In other words: it does nothing, and returns a reference to the ``View``
instance it was called on.

### Basic render implementation

Templating engines in the browser are easily interchangeable, so I am going to
assume that we are using [Handlebars][6] for most of my examples. One of the
most basic template rendering implementations that I have seen goes something
like this:

```javascript
var TemplatedView = Backbone.View.extend({
  render: function() {
    // Retrieve the text form of the template from where it has been inlined
    // in the document (typically from inside a script node):
    var templateText = $('#some-inline-script-with-a-template').text();

    // Compile the text form of the template in your template engine of choice.
    // In this case, we are using Handlebars. This step typically returns a
    // function that will accept data and return an HTML string:
    var templateFunction = Handlebars.compile(templateFunction);

    // Serialize the model into a plain object using the model's built-in
    // #toJSON implementation:
    var templateData = this.model.toJSON();

    // Call the template function with the serialized data to retrieve the
    // string representation of the rendered template:
    var htmlString = templateFunction(templateData);

    // Use the jQuery container that wraps our view's root element to set the
    // view's innerHTML to our rendered template string:
    this.$el.html(htmlString);

    // By convention, a Backbone View's render method should return a
    // self-reference:
    return this;
  }
});
```

A simple render implementation like this will get you a long way for many use
cases. Backbone relies on [jQuery#on][7] for event delegation, and always
listens on the root element that is created before your template is rendered.
This means that event handlers bound before your template is rendered will still
react to (most) events dispatched by elements rendered by your template.

### Drying things up

Usually we have many types of views in our application, which means we will
probably have several different templates. It would be nice if we didn't have to
re-implement our basic ``render`` method every time a ``View`` descendant wants
to make use of a different template.

If we re-examine our implementation above, the only thing we need to change in
each view is the value of the ``id`` attribute of the ``script`` tag that holds
the appropriate template text. We can enhance our basic implementation by having
it reference a template name that is included as part of the class definition:

```javascript
var TemplatedView = Backbone.View.extend({
  templateName: '',
  render: function() {
    // Retrieve the name of the template from a class property:
    var templateName = this.templateName;

    if (!templateName) {
      // No template, so nothing to render..
      return this;
    }

    // Retrieve the text form of the template, using the templateName value to
    // lookup a node in the DOM:
    var templateText = $('#' + templateName).text();

    // The rest remains as implemented above:
    var templateFunction = Handlebars.compile(templateText);
    var templateData = this.model.toJSON();
    var htmlString = templateFunction(templateData);

    this.$el.html(htmlString);

    return this;
  }
});
```

Given this base view class, our descendant view implementations might look
something like this:

```javascript
// Uses the template found inside the #cool-template node:
var CoolView = TemplatedView.extend({
  templateName: 'cool-template'
});

// Uses the template found inside the #hot-template node:
var HotView = TemplatedView.extend({
  templateName: 'hot-template'
});

// Skips automatic template rendering when render is called:
var SimpleView = TemplatedView.extend();
```

This implementation has two advantages over our first example: it allows a
custom template to be declared for every view that descends from
``TemplatedView``, and it allows descendant views to opt-out of automatic
template rendering by not declaring a ``templateName`` value.

## Getting real with subviews

For the purposes of this discussion, let's agree that there will be times when
you will want one view to contain another. The relationship between a view and
any subviews it contains is one of the more significant undefined relationships
in Backbone overall, and developers frequently resign themselves to manual
maintenance of subviews on a case-by-case basis.

### A naive management strategy

Here is a contrived situation where a view manually manages subviews on top of
the base Backbone framework:

```javascript
/**
 * ButtonView - A simple button class.
 */
var ButtonView = Backbone.View.extend({
  // For this view, the root element will be a button tag:
  tagName: 'button',
  // It is common for a button to be interested in a click event
  // on itself, so we declare that in our events hash:
  events: {
    'click': 'onClick'
  },
  initialize: function(options) {
    // The button label is passed in as an initialization option,
    // so we set it as a class property:
    this.label = options.label;
  },
  render: function() {
    // Our render is very simple. All we do is set the inner text of the
    // root element to the given value of 'label':
    this.$el.text(this.label);

    // Per convention, return a self-reference:
    return this;
  },
  onClick: function() {
    // For our example implementation, our only response to a click is to
    // print to the console:
    console.log('Clicked', this.label);
  }
});


/**
 * DialogView - A simple user dialog. It presents the user with a message and
 * two buttons ('Okay' and 'Cancel').
 */
var DialogView = Backbone.View.extend({
  initialize: function() {
    // We create two ButtonView instances to represent the 'Okay' and 'Cancel'
    // options that the user can use to respond to the dialog. These instances
    // are assigned to class properties:
    this.okayButton = new ButtonView({ label: 'Okay' });
    this.cancelButton = new ButtonView({ label: 'Cancel' });
  },
  render: function() {
    // First, we will set the text value of our dialog to the given value of
    // 'message' that is in the model:
    this.$el.text(this.model.get('message'));

    // Next, we need to make sure that our subviews also get rendered, so we
    // call the 'render' method on each instance:
    this.okayButton.render();
    this.cancelButton.render();

    // Finally, we append the subviews for our elements
    this.$el.append(this.okayButton.$el, this.cancelButton.$el);

    // Per convention, return a self-reference:
    return this;
  }
});
```

In the above example, we implement a ``ButtonView`` and a ``DialogView``. The
``ButtonView`` accepts a ``label`` initialization option, and the ``DialogView``
that expects a ``message`` attribute in its ``model``. With these two classes
implemented, we can now construct a basic presentation for our user:

```javascript
// Instantiate a DialogView instance with the appropriate message:
var dialog = new DialogView({
  model: new Backbone.Model({
    message: 'Are you feeling lucky, punk?'
  })
});

// Render the dialog:
dialog.render();

// Append the dialog to the body of our document:
document.body.appendChild(dialog.el);
```

Running this code on an otherwise empty document yields what you might expect: a
short text message, and two buttons. A click of each button results in an
associated message being printed to the console.

### Exploring the drawbacks

In the context of our example, manually managing the two buttons (the subviews)
of the dialog (the containing view) is not a difficult task, but as a
generalized pattern the strategy scales poorly. A large number of the
significant statements in our ``DialogView`` implementation are dedicated to
managing and rendering our subviews. As our ``DialogView`` evolves and
incorporates more subviews (for example, if we wanted our dialog to behave like
a prompt by incorporating a text input), the number of statements required to
set everything up scales linearly. Similarly, every new view that we implement
will require the same boilerplate management strategy when dealing with
subviews.

The basic implementation also suffers from a common bug related to the behavior
of Backbone's event delegation flow. The bug comes down to this: if ``render``
is ever called twice on our dialog instance (as implemented above), the 'Okay'
and 'Cancel' buttons will stop working. When we use jQuery to set the inner text
or the inner html of an element (as we do in the first line of
``DialogView#render``, and also in our earlier ``TemplatedView#render``
example), jQuery sequentially removes all children of that element. In order to
guard against memory leaks, jQuery tries to do the right thing and automatically
removes all event listeners bound to the children being removed. In the case of
our ``ButtonView``, this means our buttons are detached from the DOM and have
their ``click`` event listeners removed when render is called a second time.
However, since our render implementation reattaches the button elements so
quickly, it appears as though the buttons have mysteriously stopped behaving as
expected.

### Automating the dance

It is nice to be able to rely on automatic behavior when organizing repetitive
relationships like those between a view and its subviews. When you formalize
such behaviors, the surface area for errors shrinks to a single point in your
code. Consider the following revised ``DialogView`` implementation:

```javascript
/**
 * DialogView 2.0 - A simple user dialog. It presents the user with a message
 * and two buttons ('Okay' and 'Cancel').
 */
var DialogView = Backbone.View.extend({
  initialize: function() {
    // New in our revised class, we create a 'subviews' class property that
    // is an array. This array will hold all of this view's children:
    this.subviews = [];

    // Instead of assigning our 'ButtonView' instances as class properties, we
    // push them in order into the 'subviews' array:
    this.subviews.push(new ButtonView({ label: 'Okay' }));
    this.subviews.push(new ButtonView({ label: 'Cancel' }));
  },
  render: function() {
    // As a pre-render step, we iterate over each view that we hold a reference
    // to within our 'subviews' array:
    _.each(this.subviews, function(subview) {

      // For each subview, we call the method 'detach' on the subview's
      // jQuery-wrapped element. This method removes the view's root element
      // from the DOM without removing its event listeners:
      subview.$el.detach();
    });

    // Now that our subviews have been safely removed from the DOM, we perform
    // the main render task for this element, which is to set the inner text
    // value to our given 'message' value in our model:
    this.$el.text(this.model.get('message'));

    // As a post-render step, we iterate over each subview once more:
    _.each(this.subviews, function(subview) {
      
      // Call render on each of the subviews:
      subview.render();
      // Append the subviews to the DialogView's element:
      this.$el.append(subview.$el);
    }, this);

    // Per convention, return a self-reference:
    return this;
  }
});
```

Our revised ``DialogView`` implementation results in the same presentation for
our users, but provides a couple of advantages over the original implementation
where a developer's sanity is concerned. In our revised implementation, adding a
new subview is as simple as inserting a reference into the ``subviews`` array.
Additionally, our new implementation avoids the event delegation bug in our
original implementation by manually removing subviews before a render with
[jQuery's ``detach`` method][10].

## Tying everything together

Now that we have explored the concept of subviews a bit, let's return to the
idea of using an enhanced base class to automate behavior. We can apply this
principle to the view / subview relationship as readily as we applied it to
templates earlier. Consider the following extension to the ``TemplatedView``
class from above:

```javascript
var ContainerView = TemplatedView.extend({
  initialize: function() {
    // Our ContainerView creates a 'subviews' class property upon
    // initialization:
    this.subviews = [];
  },
  // Our ContainerView adds a helper 'addView' method to make subview operations
  // more concise:
  addView: function(view) {
    // The 'addView' method receives a 'view' reference and pushes it into the
    // 'subviews' array:
    this.subviews.push(subview);
  },
  render: function() {
    // Perform the same pre-render step from our DialogView implementation:
    _.each(this.subviews, function(subview) {
      subview.$el.detach();
    });

    // With our subviews safely moved out of the way, we call
    // 'TemplatedView#render' to perform template rendering tasks as defined
    // earlier:
    TemplatedView.prototype.render.apply(this, arguments);

    // Perform the same post-render step from our DialogView implementation:
    _.each(this.subviews, function(subview) {
      subview.render();
      this.$el.append(subview.$el);
    }, this);

    // Per convention, return a self-reference:
    return this;
  }
});
```

Our new ``ContainerView`` extends the ``TemplatedView``, and adds subview
management functionality. The bulk of the new functionality is found in the
``ContainerView#render`` method, and the new method strategically calls the
``TemplatedView#render`` method when it is safe to re-render our template.

Now that we have a generalized class for dealing with both templates and
subviews, our ``DialogView`` implementation can be made even more concise:

```javascript
/**
 * DialogView 3.0 - Templated and dry.
 */
var DialogView = ContainerView.extend({
  // Following our earlier examples, the DialogView template will look as simple
  // as this: {{ message }}. Define the appropriate template name:
  templateName: 'dialog-template',
  initialize: function() {
    // We must be careful to initialize our base class first:
    ContainerView.prototype.initialize.apply(this, arguments);

    // Taking advantage of a new 'addButton' method (below), adding new buttons
    // to our DialogView has become an extremely dry exercise:
    this.addButton('Okay');
    this.addButton('Cancel');
  },
  // We implement a helper method, again for the purpose of being concise:
  addButton: function(label) {

    // The 'addButton' method accepts a 'label' value, and creates a new button
    // with that label, adding it to the list of subviews at the same time:
    this.addView(new ButtonView({ label: label }));
  }
});
```

Our refined ``DialogView`` can take advantage of automatic template rendering
and automatic subview management. This allows it to keep its implementation
limited to the code that actually differentiates it from other views in our
application. Any bugs or design flaws that arise can be resolved in the
ancestor ``ContainerView`` and ``TemplatedView`` implementations, and the
``DialogView`` implementation will probably not have to change except for cases
of major refactoring. Now when we move on to creating new UI components, we can
rest assured that we will have reliable and useful conventions to build them on
top of.

## Closing notes, further exploration

While the conventions described above are very useful topics to consider when
developing your own application architecture, my example code should not be
considered comprehensive. There is a lot of surface area for improvement (and
even greater conciseness) that I left untouched for the purpose of brevity. I
hope that some of these topics will find their way into a future post. In the
mean time, if you are interested in exploring further, look up some of these:

 - View layouts and regions
 - Precompiled templates
 - Automatic view-model event binding
 - Non-recursive re-rendering

I hope that you found this article informative. If you come up with useful
additions to Backbone of your own, I would love to hear about them in the
comments.

[0]: http://backbonejs.org/
[1]: http://backbonejs.org/#FAQ-extending
[2]: http://backbonejs.org/docs/backbone.html#section-127
[3]: https://plus.google.com/+AngularJS/posts/aZNVhj355G2
[4]: http://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller
[5]: http://backbonejs.org/#View-delegateEvents
[6]: http://handlebarsjs.com/
[7]: http://api.jquery.com/on/#direct-and-delegated-events
[8]: http://jquery.com/
[9]: http://en.wikipedia.org/wiki/Document_Object_Model
[10]: http://api.jquery.com/detach/

