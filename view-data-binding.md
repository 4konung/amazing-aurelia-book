# View data binding

## Binding

From [how-aurelia-works](https://www.danyow.net/how-aurelia-works/)

A good place to start is in the templating module, with a component called the `ViewCompiler`. The `ViewCompiler`'s job is to compile views into a `ViewFactory` which will be used to instantiate instances of your templates, called `Views`. Some view factories will be used to instantiate only one instance of a particular template. The `ViewFactory` that resulted from compiling your `app.html` template typically falls into this category. Other view factories for templates used with repeat.for may be used to instantiate tens, hundreds or even thousands of `View` instances.

A `ViewFactory` is comprised of:

- a template (a document fragment)
- a set of HTML behavior instructions (instructions for creating custom elements or custom attributes that appear in the template)
- binding expressions (factories for creating bindings that appear in the template)
- dependencies (thing's you've `<require from="...">`)
- and more...

When Aurelia loads one of your HTML templates the markup is parsed by the browser into a document fragment. The document fragment is the browser's object representation of your HTML. It's a tree structure and all the node names and attribute names have been lower-cased by the browser's HTML parser (except for SVG elements). The `ViewCompiler` traverses each DOM node in the tree and checks whether the node's name matches the name of a custom element. 

For now we won't go into the details of what happens when a custom element is discovered because it doesn't really matter to the binding system whether an element is custom or built-in. In addition to checking for custom elements, the `ViewCompiler` asks the `BindingLanguage` implementation to examine each node's attributes and content.

### BindingLanguage

The standard `BindingLanguage` implementation that ships with Aurelia looks for attributes whose name ends with `.bind`, `.one-way`, `.delegate`, `.trigger` or starts with `ref`, etc. These attribute postfixes and prefixes are known as "binding commands".

The standard `BindingLanguage` also checks the element's content for string interpolation expressions. When a binding command or string interpolation is discovered, the attribute's `value` (text) or interpolation's `content` (also text) is sent to the binding system's `Parser`. 

### Parser

If you've written your binding expression correctly the parser will be able tokenize the text (split it into a series of interesting parts) and convert the tokens to an abstract syntax tree (AST). The AST is the object representation of your JavaScript binding expression. We'll dive into the AST in a minute. What's important to understand now is by this point the `ViewCompiler` has gathered a whole bunch of information about the binding expression in your template. It knows the name of the DOM element, the name of the element's attribute you intend to bind to (if applicable), the binding command (bind/trigger/etc) and it has the AST representation of your JavaScript binding expression. All of these parts are used to construct a `BindingExpression` which is a factory for creating binding instances.

Once the `ViewCompiler` has completely traversed the document fragment it returns the `ViewFactory` instance which will get cached until it's needed. 

### CompositionEngine

Aurelia compiles templates on an as-needed basis so it's likely to be used soon after it's compiled. Orchestrating all the work described above and the steps to follow is the `CompositionEngine` in conjunction with a `Controller`, which are parts of the templating module. The `CompositionEngine` is a high level component that creates controllers and executes composition instructions. Composition instructions are the result of application bootstrapping, routing and the `<compose>` element. They tell Aurelia to compose a view with a view-model. 

### Controller

The `Controller` is a little bit lower level. It "owns" the View, and ViewModel (VM) and takes us through the created, bind, attached, detached and unbind composition lifecycle events that occur when composing a View and VM, injecting the View into the DOM and later removing it when it's no longer needed. Let's go through these steps, one-by-one, focusing on what happens with respect to data-binding.

The `ViewFactory`'s create method will be called to create a `View` instance. A `View` in this sense is a JavaScript class with methods for all of the composition lifecycle events. Creating a `View` involves several steps. First the `ViewCompiler`'s document fragment is cloned to create the element that will ultimately be injected into the DOM and data-bound. This is the element that's given to your VM's constructor when you use `@inject(Element)`. Next, the `ViewCompiler` will execute all it's instructions: creating custom element instances, custom attribute instances, and binding instances.

Creating a binding instance is accomplished by invoking the `createBinding` method on each `BindingExpression`, passing in the specific DOM element, custom element or custom attribute that is relevant to the `BindingExpression`.

The `createBinding` method uses the `ObserverLocator` to locate the appropriate property observer for the combination of DOM element and property. Property observers expose a `subscribe`, `unsubscribe`, `getValue` and `setValue` interface. Each property observer has a specific observation strategy tailored for certain types of objects or elements and their various properties.

For example, if you were binding an input element's value attribute there's an observer implementation that will subscribe to the input's `change` and `input` DOM events. With the property observer in hand `createBinding` will instantiate a `Binding` instance, passing the property observer to the `Binding`'s constructor. This observer is known as the `Binding`'s `targetObserver`. A few other items are passed to the `Binding`'s constructor: the DOM element, known as the binding's target, the attribute name, the binding mode (`one-way`, `two-way` or `one-time`) and last but not least, the `BindingExpression`'s AST, which is known as the binding's `sourceExpression`.

Once the `ViewFactory`'s create method has finished executing all the instructions and creating all the bindings it will instantiate the View, whose constructor will receive the DOM element, bindings, controllers, and a few other items. With the `View` created the `Controller` can execute the view's bind method, passing in the binding context and override context. The binding context and override context tuple is known as the `scope`. The binding context is usually the view-model and the override context is an object that stores additional properties you can bind to, depending on the situation. Some examples include: `$event`, `$this`, `$parent`, `$index`, `$odd`, `$first`, etc.

The view's `bind` method loops through all of it's binding instances and calls their bind method, passing in the binding context and override context. This is where the AST in the binding's `sourceExpression` becomes important. The abstract syntax tree is the heart of the binding system. It's the object representation of your binding's JavaScript expression. 

Each node in the tree is an implementation of a very binding-specific interface. There are about 20 different AST node types, each tailored to a particular type of JavaScript expression. The AST can be evaluated by calling the root AST node's `evaluate` method and passing in the scope. Each node in the AST knows how to evaluate it's piece of the expression using the scope and the end result will be the value of the JavaScript expression. 

After the binding gets the model value by evaluating the `sourceExpression` it assigns this value to the view by calling the `targetObserver`'s `setValue` method. Next the binding will check it's binding mode. If it's `one-time`, there's nothing left to do. If it's `one-way` or `two-way` the binding will use the AST's connect method to subscribe to changes in the view-model. Each node in the AST knows which view-model properties to observe and will use the `ObserverLocator` to create property observers to subscribe to property change events. Finally, if the binding mode is `two-way` the binding will call the `targetObserver`'s `subscribe` method.

At this point the view and view-model are data-bound. When changes occur in the model, the property observers created when the AST was connected will fire change events, triggering the binding to update the target by calling `targetObserver.setValue`. When changes occur in the view the property observer known as the `targetObserver` will trigger the binding to update the source by calling `sourceExpression.assign(scope, value)`. All that remains is for the `Controller` to attach the view to the DOM. Later, when the view is no longer needed it will be detached from the DOM and `unbind` will be invoked, unbinding all the views, which will unsubscribe all the property observers.