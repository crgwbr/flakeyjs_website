## Flakey.controllers

#### Controllers
Flakey implements a fairly standard Model-View-Controller architecture, similar to Spine.js, Backbone.js, or Ruby on Rails. *Controllers*, and their similar cousin *Stacks*, are how a Flakey app controls what is displayed on the screen and how user actions are routed to event functions. *Views*, at least from Flakey's perspective are just HTML templates. Flakey can use any HTML templating system, but in these example's we'll use [Eco](https://github.com/sstephenson/eco) since support for Eco is built into Flakey via Flakey.templates.

The first step in creating a controller is to implement the `Flakey.controllers.Controller` class and provide a constructor function.

    class NoteSelector extends Flakey.controllers.Controller
      constructor: (config) ->
        @id = 'note-selector'
        @class_name = 'note-selector view'
        @actions = {
          'click li.note': 'select_note'
        }
        super(config)
        
        @append ExampleController
        
        @tmpl = Flakey.templates.get_template('selector', require('./templates/selector'))
      
      render: () ->
        context = {
          selected: @query_params.id,
          notes: Note.all()
        }
        @html @tmpl.render(context)
      
      select_note: (event) =>
        @container.children('ul').children('li.note').removeClass('selected')
        selected_note = $(event.target)
        selected_note.addClass('selected')
        Flakey.util.querystring.update({id: selected_note.attr('id')}, merge = true)
        event.preventDefault()
        
In this example, we create a new controller called `NoteSelector`. Within the constructor method, we set several important attributes:

- `@id` corresponds to the id attribute on the controller's div element.
- `@class_name` corresponds to the className attribute on the controller's div element.
- `@actions` is a map where the key is a JQuery action and element selector separated by a space. The value is a string containing the name of the method to call when the event is triggered.
- `@append` is a useful feature that allows us to attach another controller to this controller. Whenever the NoteSelector controller is made active, so are any appended controllers; the same goes for when NoteSelector is made passive.

Additionally, to save processing time in the future, the constructor get's the Eco template we'll use for this controller and save it as `@tmpl`. The render method is the only other method your controller has to implement. It shouldn't take any parameters and doesn't need to return anything; the @render method's only purpose is to render HTML and then modify the controller's div attribute's html. This is most easily accomplished by calling the built-in @html() method and passing in the desired html string. In the case above, the render function builds a context object, renders HTML using the template we got earlier, and passes it into @html().

The @select_note() method above is an event handler, since we set it as the callback event to the 'click li.note' action in the constructor. Because of that, you'll notice several unique things about @select_note():

1. It accepts the event parameter. This is a standard JQuery event object containing useful information like the target element and action.
2. It uses the `=>` function definition instead of `->`. Using a fat arrow for function definition in CoffeeScript is similar to JQuery's $.proxy() function, in that it ensures @select_note() will *always* be called with it's parent object as its lexical scope. This isn't strictly *necessary* for event handler functions, but without it, inside the function `@` or `this` will point to the event's target element rather than an our controller object. Since we need to access attributes of the controller object, we use a fat arrow to retain it's scope.

#### Active and Passive states
Controllers are enabled/disabled by calling controller.make_active() and controller.make_passive(), respectively. Its very important to note that Flakey never actually sets CSS properties to show/hide an element. Instead it simply sets either an active or passive class on the container. Additionally, controller.make_active() calls your controllers render method.

#### Stacks
Stacks are similar to controllers in that they let you control what is displayed on the screen. Stacks, however, don't directly manage content. Instead they manage a stack of separate controllers to ensure that only one s visible at the same time.

    class MainStack extends Flakey.controllers.Stack
      constructor: (config) ->
        @id = 'main-stack'
        @class_name = 'stack'
        @controllers = {
          'notes': NotesController
          'contacts': ContactsController
        }
    
        @routes = {
          '^/notes$': 'notes',
          '^/contacts$': 'contacts'
        }
        @default = '/notes'
    
        super(config)

The most important parts of this example are the @controllers and @routes objects in our new stack. The @controllers attribute defines aliases for each of the controller classes to be managed by our stack. Notice we're directly passing in controller *classes*. There is no need to instantiate them, since the stack will do that for us. Secondly the @routes attribute defines a series of regular expressions tied to our controllers. When the stack is active, these regexps are checked against the browser's URL hash (window.location.hash). The matching controller (of the default fallback) is then made active, while the rest of the managed controllers are made passive. This is repeated anytime the URL hash changes.

#### Starting off our app
Now that we have controllers defined, we need to instantiate the top-level controller/stack.

    $(document).ready () ->
      settings = {
        container: $('body')
      }
      Flakey.init(settings)
  
      # Sync models
      Flakey.models.backend_controller.sync('Note')
      
      note_app = window.note_app = new MainStack()
      note_app.make_active()
    
Most of this we've seen already; what's important to this tutorial are the last two lines. That makes a new instance of MainStack and marks it active. In this case, since it's a stack, it will try to find a controller corresponding to the current URL hash. If it doesn't find one, it will set the hash to @default, then will check again. Whichever controller it finds will be marked as active, while the rest are marked as passive.




