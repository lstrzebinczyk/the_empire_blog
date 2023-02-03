## The plan

There are, quite clearly, 2 different user interfaces in this app. World map is the first one. How to draw things clearly and understandably in there is a problem for another time.
THe other one is everything else. We need to build a basic user interface (UI), which will show data and allow for interaction. For web apps, there's html and css. Since we don't have that, we need to build one from scratch, based on text and rectangles.

I will present a progression from an idea, through implementation, to a reasonable working system through several next posts, but don't let it fool you. I've spent quite a bit of time and went through multiple iterations before I had something usable and presentable. This is a very hard problem to solve, and what you will not see in here is all the ideas that turned out not to work, once there is more than the most basic cases.

Building this from scratch gave me new appreciation for html, and other frameworks for organizing UIs.

Allrighty then, first we need to analyze...

## What we have

And that's mostly the button component:

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

Visually, it's a rectangle of a specific color, with some text centered vertically and horizontally. In code, every time we draw it, we rebuild and reposition both the background, and the text.

In addition to that, we have the bottom menu:

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
Button positions are hardcoded, and the background is recalculated on each render.

## What we want

Major issue with this approach is having to manually specify position for everything. This system must be flexible and allow me to change things around, experiment. What if we try bigger gap between the buttons? Or make the buttons wider, or less wide, or less height? All of these ideas would require me to recalculate pixel positions for each button. That's not acceptable.

I want to *declare* what I want, and have a system which will figure out details for me.

Let's go through what I want:
- I want to tell things to be centered inside an area
- I want to be able to tell things to render vertically and horizontally, one after another
- I want that to be built as a composable, reusable set of components.

We'll deal with the first point (and the third, by necessity) today, and with the second point next time.

## UI::Item

This time, I'll do it a bit backwards: I'll by thinking about the building blocks of the system, and we'll then compose them to improve the implementation of `UI::Button`.

First, we need a single, unified idea of what a UI Component is. I'll introduce a `UI::Item` module, and all UI components will have to include it:

```crystal
# src/lib/ui/_item.cr

module UI
  module Item
    # all ui items must be drawable
    include SF::Drawable

    # all ui components must be able to move
    abstract def position=(new_position : Tuple(Int32, Int32))
    # all ui components take a specific area of screen, and they must be able to tell what it is
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

    # If there is a background, draw it
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

## UI::Box

We need a parent UI component. One that we will be able to tell: "Here is an area of the screen and a set of requirements. Figure it out.".
To make things simple, that parent component will be allowed to have a single child, and will render it centered vertically and horizontally within itself.

Let's go through the implementation:

```crystal
# src/lib/ui/containers/box.cr

module UI
  class Box
    # The box itself is a ui component
    include UI::Item

    # This is the single child component. We'll allow it to be nil.
    @renderable : UI::Item | Nil

    # It's expected to get bounding rectangle in the argument, and will expose it
    getter bounding_rectangle : SF::IntRect

    # Other than @bounding_rectangle, expect a block and call that black with itself
    # We will expect that block to set @renderable
    def initialize(@bounding_rectangle, &block : UI::Box -> UI::Item)
      @renderable = nil
      block.call(self)
    end

    # UI::Item forces us to give it a `#position=` method.
    # If new position is given to the parent, we must reposition the child as well
    def position=(new_position)
      @bounding_rectangle.position = new_position
      reposition_renderable!
    end

    # Nothing interesting happens here, `super` draws the background, then draw the child component
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

    # Initializer receives a block, and we will expect that block to set @renderable.
    # This method is how it does so. It sets the renderable, and makes sure the positioning is correct.
    def add_renderable(renderable : UI::Item)
      @renderable = renderable
      reposition_renderable!
    end

    # Center the child component horizontally and vertically
    # We figured out the logic previously inside the UI::Button, and can now move it here.
    # Notice that for this to work, the child components must have `#bounding_rectangle` and `#position=` methods.
    # All UI::Item components must have them, so we can use any UI::Item component as a child
    # That is, in fact, why we needed UI::Item to implement them.
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

Alright, we have a box and a concept of UI item. Since we are aiming at making the button implementation simpler, the missing piece is ...

## UI::Text

A text component. A fairly simple wrapper over `SF::Text`. Everything it needs is already implemented inside `UI::Button`, we just need to extract it:

```crystal
# src/lib/ui/text.cr

module UI
  class Text
    include UI::Item

    # I have decided all child components will receive their parent in the constructor. Comes in handy.
    # Other than that it's straightforward, take `string`, build a `@text`.
    def initialize(@parent : UI::Box, string : String)
      @text = SF::Text.new(string, Constants::FONT, 60)
      @text.color = SF::Color::Black
    end

    # Reuse `@text.global_bounds` for bounding rectangle.
    # That `#to_i` method isn't there, but it's easy enough to add : )
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

    # Render the background, render the text
    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      super(target, states)

      target.draw(@text, states)
    end
  end
end
```

Easy-peasy. And finally, we want the `UI::Box` to be able to create `UI::Text` and set it as a children. We'll do it like so:

```crystal
# src/lib/ui/containers/box.cr, class UI::Box

def text(**args)
  text = UI::Text.new(self, **args)
  add_renderable(text)

  text
end
```

That's a lot of setup, but we're at the finishing line. All of this allows us to reorganize the...

## UI::Button

Once again, there is a lot to unpack here. Let's check the new implementation of `UI::Button`, and then I'll explain the interesting bits:

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

First thing to notice is that this `UI::Button` is not a `UI::Item`. It will be later. For now we are only interested in reorganizing it's internals with what we introduced.

Second, this is the grand finale:
```crystal
private def build_ui
  UI::Box.new(@bounding_rectangle) do |c|
    c.text(string: @text)
  end
    .background(fill_color: fill_color())
end
```

It's our new UI framework in working! It's a `UI::Box`, it receives a block which sets a `UI::Text` as it's child, we give it a `@bounding_rectangle` in constructor, and a background.
Additionally, instead of calculating the UI on every render, we'll create it in constructor, save it, and render the saved variable.

Nothing changed visually, but we implemented important architecture!

Great success. And since we can now implement text inside a box, it should be enough to build...

## The right menu

We want the right menu to present information related to the active mode. Currently right menu isn't even aware of the active mode, so we'll start with that:

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

  # It's a situation similiar to what we did in the `UI::Button`,
  # except we want the `@active_mode` to produce a `@ui` for the right menu.
  @ui = @active_mode.right_menu(@bounding_rectangle)
end

# Whenever active mode changes, we'll need a new UI
def active_mode=(new_active_mode)
  @active_mode = new_active_mode
  @ui = @active_mode.right_menu(@bounding_rectangle)
end

# Render the new @ui
def draw(target : SF::RenderTarget, states : SF::RenderStates)
  target.draw(@ui, states)
end
```

So the only missing piece is actually implementing the ui-generating function inside modes. Let's just copy what we did for buttons:

```crystal
# src/the_empire/mode/move_mode.cr, class TheEmpire::Mode::MoveMode

# we'll accept a bounding rectangle and return a ui component encompassing that rectangle, with a background and a text in the middle
def right_menu(bounding_rectangle : SF::IntRect)
  UI::Box.new(bounding_rectangle) do |c|
    c.text(string: "Move")
  end
    .background(fill_color: Constants::COLOR::MENU::BACKGROUND)
end
```

We'll also give a respective method to `POIMode`.

And voila! We have useful information on screen:

<video width="100%" src="/the_empire_blog/docs/assets/posts/9/working_right_menu.mp4" controls autoplay></video>

Great success, we click on screen and things happen.

In the next one, we'll handle more use case by introducing [vertical, horizontal and spacer](10-vertical-horizontal-and-spacer.html)!
