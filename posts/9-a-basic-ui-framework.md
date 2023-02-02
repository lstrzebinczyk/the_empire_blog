## The plan

There are, quite clearly, 2 different user interfaces in this app. World map is the first one. It will h how to draw things in a clear, understandable manner. That is a problem for some oave an interesting problems related tother time.

The other user interface is much more mundane and basic: we need to present data and allow for basic interaction. In a web application, we would use html, but since we don't have that, we need to build one from more basic building blocks.

Now, I want to note this: I will present a quite clear progression from idea, through implementation, all the way to a working system. Though it might look like a straightforward, the system I originally implemented went through multiple phases and ideas for every aspect of it. Building a UI framework is hard, and trying to solve some of the issues myself gave me a new appreciation for the technologies widely used for this purpose.

Let's start this by checking...

## What we have

First, we have a button component. Let's see:

```crystal
# src/lib/ui/button.cr, class UI::Button

def draw(target : SF::RenderTarget, states : SF::RenderStates)
  background = SF::RectangleShape.new(@bounding_rectangle)
  if self.active
    background.fill_color = SF::Color.new(249, 180, 75)
  else
    background.fill_color = SF::Color.new(142, 142, 142)
  end

  text = SF::Text.new(@text, Constants::FONT, 60)
  text.color = SF::Color::Black
  text_position_x = @bounding_rectangle.left + @bounding_rectangle.width / 2 - text.local_bounds.width / 2 - text.local_bounds.left
  text_position_y = @bounding_rectangle.top + @bounding_rectangle.height / 2 - text.local_bounds.height / 2 - text.local_bounds.top
  text.position = { text_position_x, text_position_y }

  target.draw(background, states)
  target.draw(text, states)
end
```

Visually, it's a rectangle of a specific color, with some text sentered exactly in the middle.

Second, we have the bottom menu:

```crystal
# src/the_empire/bottom_menu.cr, class TheEmpire::BottomMenu

def initialize(position, size, @modes, @active_mode, @omit_event)
  @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])

  @buttons = @modes.map_with_index do |mode, index|
    UI::Button.new(
      position: { @bounding_rectangle.position[0] + 20 + index * 220, @bounding_rectangle.position[1] + 20},
      size: {200, 80},
      text: mode.button_text,
      active: mode == @active_mode,
      on_click: -> {
        event = TheEmpire::Event::ChangeModeEvent.new(mode)
        @omit_event.call(event)
      }
    )
  end
end

def draw(target : SF::RenderTarget, states : SF::RenderStates)
  background = SF::RectangleShape.new(@bounding_rectangle)
  background.fill_color = Constants::COLOR::MENU::BACKGROUND

  target.draw(background, states)

  @buttons.each do |button|
    target.draw(button, states)
  end
end
```

Visually, we have a rectangle with a specific color, and a set of buttons rendered horizontally, aligned to the right, and centered vertically.

## What we want

Major issue with this approach is having to manually specify position for everything. I definitely don't want to calculate pixels every time I add something to the ui.
And I most definitely don't want to rething how to center things, or align to left/right every time I add new ui items.

I want to *declare* what I want, and have a system which will figure out where to put things.

So how would a better system look like?
- I want to *declare* what I want, and have a system which will figure out where to put things.
- I want a component, which will accept a single child component and center it
- I want a component, which will accept a collection of children and render them horizontally one after another. Another one for vertical rendering, but that's for later.
- I want my horizontal component to be able to align children to the leftmost available area
- Finally, I want all of this toolset to be composable and reusable

## UI::Item

So how would a buttons implementation look like? I like a structure like this:

```crystal
UI::Box.new(@bounding_rectangle) do |c|
  c.text(string: @text)
end
  .background(fill_color: fill_color())
```

From the code above:
- `UI::Box` is a component which takes a bounding box (and thus an area on the screen), takes a single component, and centers that component inside itself.
- `UI::Box` initialization takes a block, and by executing that block we create a sub-component and storing it inside `UI::Box`
- `c.text(string: @text)` creates a sub-component, which simply renders a text
- `.background(fill_color: fill_color())` updates the background color of the `UI::Box`.

The `UI::Box` should be able to take any ui component: a `text`, `button`, anything at all. To achieve that, we need a way to represent a concept of visual component in the code.
We'll do it with a module. It will be expected to be included in all classess representing UI components.

** snaps fingers **

```crystal
# src/lib/ui/_item.cr

module UI
  module Item
    # since all items must be drawable, it makes sense to declare it here
    include SF::Drawable

    # this will force all ui components to deal with being told to move
    abstract def position=(new_position : Tuple(Int32, Int32))
    # all ui components must have a position on screen
    abstract def bounding_rectangle : SF::IntRect

    # all ui components can have a background
    @fill_color : SF::Color | Nil

    # all ui components will need to deal with events. This makes them all do nothing by default
    def handle_event(event)
      nil
    end

    # set background to that color, please
    def background(fill_color : SF::Color)
      @fill_color = fill_color

      self
    end

    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      if fill_color = @fill_color
        background = SF::RectangleShape.new(bounding_rectangle)
        background.fill_color = fill_color

        target.draw(background, states)
      end
    end
  end
end
```

## UI::Text

Everything we need for a text component is already implemented in the `UI::Button`. We can easily extract it and implement a simple `UI::Text`:

```crystal
# src/lib/ui/text.cr

module UI
  class Text
    # That module we created above. It says "I am a ui component"
    include UI::Item

    # All UI is now going to be built with this new system, and therefore UI::Box will be the only object passed around.
    # All ui components will be parts of a components tree, and therefore there will always be a parent container.
    # I have decided that we will be passing that container into all child components.
    # It comes in handy
    def initialize(@parent : UI::Box, string : String)
      @text = SF::Text.new(string, Constants::FONT, 60)
      @text.color = SF::Color::Black
    end

    # This is required by the UI::Item.
    # Since we have only 1 element of internal structure, we can reuse it for bounding rectangle.
    def bounding_rectangle : SF::IntRect
      @text.global_bounds.to_i
    end

    # This logic comes directly from the UI::Button, and text displacement logic we discussed before
    def position=(new_position)
      text_bounds = @text.local_bounds
      text_top = text_bounds.top
      text_left = text_bounds.left

      position_x = (new_position[0] - text_left).to_i
      position_y = (new_position[1] - text_top).to_i

      @text.position = {position_x, position_y}
    end

    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      # this renders the background, should it have some
      super(target, states)

      target.draw(@text, states)
    end
  end
end
```

Allright, awesome.

Now, the meaty part:

```crystal
# src/lib/ui/containers/box.cr

module UI
  class Box
    # The box is itself a ui component
    include UI::Item

    # We expect this component to just have 1 child component.
    @renderable : UI::Item | Nil

    getter bounding_rectangle : SF::IntRect

    # Save the @bounding_rectangle, expect a block and call that black with itself
    def initialize(@bounding_rectangle, &block : UI::Box -> UI::Item)
      @renderable = nil
      block.call(self)
    end

    # Main job of this component is to hold another component. Now, when it's asked to change position, it must move itself
    # but also make sure that the child components position remains correct
    def position=(new_position)
      @bounding_rectangle.position = new_position
      reposition_renderable!
    end

    # Nothing interesting happens here, draw the background, draw the child component
    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      super(target, states)

      if item = @renderable
        target.draw(item, states)
      end
    end

    # If there is a child component, delegate all events to the child component
    def handle_event(event : SF::Event)
      if item = @renderable
        item.handle_event(event)
      end
    end

    # This is how we will add text component as own child.
    # We could implement it simpler, yes, but that structure will come in handy in next post.
    def text(**args)
      text = UI::Text.new(self, **args)
      add_renderable(text)

      text
    end

    # That's also quite plain. Set the component as new child component and reposition it.
    def add_renderable(renderable : UI::Item)
      @renderable = renderable
      reposition_renderable!
    end

    # After setting the child component and changing position of the box, we want to make sure the child component is precisely in the middle
    # We figured out the logic previously, and can now move it here.
    # Notice that we expect the child components to have `#bounding_rectangle` and `#position=` methods
    # Since all UI::Item components are forced to implement those, we can use any component that includes that module as a child component in the box
    # That is, in fact, why we needed UI::Item to implement these methods.
    def reposition_renderable!
      if item = @renderable
        item_width = item.bounding_rectangle.width
        item_height = item.bounding_rectangle.height

        x_position = (bounding_rectangle.left + bounding_rectangle.width / 2 - item_width / 2).to_i
        y_position = (bounding_rectangle.top + bounding_rectangle.height / 2 - item_height / 2).to_i

        item.position = {x_position, y_position}
      end
    end
  end
end
```

There's a lot to unpack here, but that allows us to change buttons implementation to the following:

```crystal
# src/lib/ui/button.cr

module UI
  class Button
    include SF::Drawable

    @bounding_rectangle : SF::IntRect
    @text : String
    @on_click : Proc(Nil)
    @ui : UI::Item

    property active : Bool = false

    def initialize(position, size, @text, @on_click, @active = false)
      @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])
      @ui = build_ui()
    end

    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      target.draw(@ui)
    end

    def handle_event(event : SF::Event::MouseButtonPressed)
      if @bounding_rectangle.contains?(event.x, event.y)
        @on_click.call
      end
    end

    private def build_ui
      UI::Box.new(@bounding_rectangle) do |c|
        c.text(string: @text)
      end
        .background(fill_color: fill_color())
    end

    private def fill_color
      if @active
        SF::Color.new(249, 180, 75)
      else
        SF::Color.new(142, 142, 142)
      end
    end
  end
end
```

We now have a cached renderable `@ui` component, which is underneath a complex composition of components! And we no longer have to manually position the text inside the background, it happens automatically.
Nothing changed visually, but we implemented important architeture!

We can now implement text inside a box. This should be enough to build...

## The right menu

We want the right menu to present us information related to the active mode. Currently right menu isn't even aware of the active mode, so we'll start with that:

```crystal
# src/the_empire.cr, class TheEmpire#initialize

@right_menu = TheEmpire::RightMenu.new(
  position: {@window_width - RIGHT_MENU_WIDTH, 0},
  size: {RIGHT_MENU_WIDTH, @window_height},
  active_mode: @active_mode # <- this is new
)
```

And we will have to update the active mode on `@right_menu` whenever active mode changes:

```crystal
# src/the_empire.cr, class TheEmpire

def handle_page_event(event : TheEmpire::Event)
  case event
  when TheEmpire::Event::ChangeModeEvent
    @active_mode = event.mode
    @bottom_menu = TheEmpire::BottomMenu.new(
      position: {0, @window_height - BOTTOM_MENU_HEIGHT},
      size: {@window_width - RIGHT_MENU_WIDTH, BOTTOM_MENU_HEIGHT},
      modes: @modes,
      active_mode: @active_mode,
      omit_event: ->handle_page_event(TheEmpire::Event)
    )
    @right_menu.active_mode = @active_mode # <- this is new
  end

  nil
end
```

`RightMenu` doesn't support any of this, but it is easily fixable:

```crystal
# src/the_empire/right_menu.cr, class TheEmpire::RightMenu

# we declare that we're now going to store a mode object
@active_mode : TheEmpire::Mode::BaseMode
# we are also going to store the ui
@ui : UI::Item

# save it during initialization
def initialize(position, size, @active_mode)
  @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])
  # call the method we just implemented on the modes to give us a ui
  @ui = @active_mode.right_menu(@bounding_rectangle)
end

# and have a method to update it
def active_mode=(new_active_mode)
  @active_mode = new_active_mode
  # whenever active mode changes, create a new ui
  @ui = @active_mode.right_menu(@bounding_rectangle)
end

# use the new @ui for rendering
def draw(target : SF::RenderTarget, states : SF::RenderStates)
  target.draw(@ui, states)
end
```

So the only missing piece is actually implementing the ui-generating function inside modes. For now, we'll just copy what we did for buttons:

```crystal
# src/the_empire/mode/move_mode.cr, class TheEmpire::Mode::MoveMode

# we'll accept a bounding rectangle and return a ui component encompassing that rectangle, with a text in the middle and a background
def right_menu(bounding_rectangle : SF::IntRect)
  UI::Box.new(bounding_rectangle) do |c|
    c.text(string: "Move")
  end
    .background(fill_color: Constants::COLOR::MENU::BACKGROUND)
end
```

Let's recap:
- Both modes now have a `#right_menu` method, which accepts a bounding rectangle, and returns a `UI::Item` object
- Right menu now follows what the active mode is
- When right menu is created, and whenever active mode changes, a ui is generated for the right menu with the `#right_menu` method

And voila! We have useful information on screen:

<video width="100%" src="/the_empire_blog/docs/assets/posts/9/working_right_menu.mp4" controls autoplay></video>

Great success, we click on screen and things happen. We need that ui system to handle several more use cases, so in the next one,
we'll introduce [vertical, horizontal and spacer](10-vertical-horizontal-and-spacer.html)!
