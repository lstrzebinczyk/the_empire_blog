## Current status

Let's go through everything we have in place for events management, as provided by the [CrSFML](https://oprypin.github.io/crsfml/) library.

We have a main loop, an indefinitely-running piece of code, which runs in loop:

```crystal
# src/main.cr

require "./the_empire"

the_empire = TheEmpire.new

while the_empire.running?
  the_empire.handle_events
  the_empire.update
  the_empire.render
end
```

From this, the part we are interested in is `the_empire.handle_events`. In every loop we process, we will attempt to process the events:

```crystal
# src/the_empire.cr, class TheEmpire

def handle_events
  while event = @window.poll_event
    handle_event(event)
    @world_map.handle_event(event)
  end
end
```

We are calling `@window.poll_event`, for as long as long it returns values. For each event we get, we call `handle_event(event)` and `@world_map.handle_event(event)`.
The first of these is defined like this:

```crystal
# src/the_empire.cr, class TheEmpire

def handle_event(event : SF::Event::Closed)
  @window.close
end

# Ignore any other event
def handle_event(event)
end
```

So its only job is to close the window if `SF::Event::Closed` event comes along. `@world_map.handle_event(event)` uses all remaining events to move the map.

Wait. Like... All of them?

<video width="100%" src="/the_empire_blog/docs/assets/posts/7/a_problem.mp4" controls autoplay></video>

Oh boy, yeah, so that's a problem.

What is happening here is that we are clicking and moving mouse around in right menu and bottom menus, but those clicks and moves are being applied to the world map.

That's not the behavior we are expecting. What we want is this: If I press on `WorldMap`, `WorldMap` should react, and if I press on `BottomMenu`, `BottomMenu` should react.
Those shouldn't mix.

I'll update the events handling code to look like this:

```crystal
# src/the_empire.cr, class TheEmpire

def handle_events
  while event = @window.poll_event
    handle_event(event)
  end
end

# Handle the close event specifically
def handle_event(event : SF::Event::Closed)
  @window.close
end

def handle_event(event)
  @world_map.handle_event(event)
  @bottom_menu.handle_event(event)
  @right_menu.handle_event(event)
end
```

`#handle_events` now picks the event from `@window` and passess them along. `#handle_event` closes the window, if that event is `SF::Event::Closed`, but otherwise delegates the event to all three of `@world_map`, `@bottom_menu` and `@right_menu`. Those classess will now accept all events and will need to decide whether or not they should react to them.

`BottomMenu` and `RightMenu` will get an empty method for now:

```crystal
def handle_event(event)
end
```

This is how `WorldMap` handles mouse events currently:

```crystal
# src/the_empire/world_map.cr, class TheEmpire::WorldMap

def handle_event(event : SF::Event::MouseButtonPressed)
  @moving_around = true
  @mouse_button_initial_x = event.x
  @mouse_button_initial_y = event.y
end

def handle_event(event : SF::Event::MouseButtonReleased)
  @moving_around = false
end

def handle_event(event : SF::Event::MouseMoved)
  if @moving_around
    x_delta = @mouse_button_initial_x - event.x
    y_delta = @mouse_button_initial_y - event.y

    move({x_delta, y_delta})

    @mouse_button_initial_x = event.x
    @mouse_button_initial_y = event.y
  end
end
```

We want to make sure that this processing only happens if mouse is over the `WorldMap`. We might do it like like this:

```crystal
# src/the_empire/world_map.cr, class TheEmpire::WorldMap

def handle_event(event : SF::Event::MouseButtonPressed)
  if @bounding_rectangle.contains?(event.x, event.y)
    @moving_around = true
    @mouse_button_initial_x = event.x
    @mouse_button_initial_y = event.y
  end
end

def handle_event(event : SF::Event::MouseButtonReleased)
  if @bounding_rectangle.contains?(event.x, event.y)
    @moving_around = false
  end
end

def handle_event(event : SF::Event::MouseMoved)
  if @bounding_rectangle.contains?(event.x, event.y)
    if @moving_around
      x_delta = @mouse_button_initial_x - event.x
      y_delta = @mouse_button_initial_y - event.y

      move({x_delta, y_delta})

      @mouse_button_initial_x = event.x
      @mouse_button_initial_y = event.y
    end
  end
end
```

`@bounding_rectangle` is, of course, a rectangle, and `@bounding_rectangle.contains?(event.x, event.y)` will tell is whether or not the events point is contained by it. Neat.

Does it work? It does!

<video width="100%" src="/the_empire_blog/docs/assets/posts/7/clicks_contained.mp4" controls autoplay></video>

Does it, though?

<video width="100%" src="/the_empire_blog/docs/assets/posts/7/new_problem.mp4" controls autoplay></video>

If I press the mouse, move outside of the `WorldMap`, release the mouse and move back to the `WorldMap`, we have ended in a state where the mouse is not pressed, but map is being dragged behind the mouse.
What I would **really** like, is to have `MouseEnter` and `MouseLeave` events for each container, but we don't have that. So what I'll do, is this:

```crystal
# src/the_empire/world_map.cr, class TheEmpire::WorldMap

def handle_event(event : SF::Event::MouseButtonPressed)
  if @bounding_rectangle.contains?(event.x, event.y)
    @moving_around = true
    @mouse_button_initial_x = event.x
    @mouse_button_initial_y = event.y
  end
end

def handle_event(event : SF::Event::MouseButtonReleased)
  @moving_around = false
end

def handle_event(event : SF::Event::MouseMoved)
  if @moving_around
    x_delta = @mouse_button_initial_x - event.x
    y_delta = @mouse_button_initial_y - event.y

    move({x_delta, y_delta})

    @mouse_button_initial_x = event.x
    @mouse_button_initial_y = event.y
  end
end
```

Begin drag-and-drop if mouse is over the `WorldMap`, but process and finish it irregardless of position.
That will work for now. There is still handling of `AWSD` keys and mouse scroll. Those will ignore the position boundary, but that's an issue for another day.

And that leaves us with the final missing piece: handling the button. We'll update the `BottomMenu` like so:

```crystal
# src/the_empire/bottom_menu.cr

class TheEmpire
  class BottomMenu
    include SF::Drawable

    def initialize(position, size)
      @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])
      @button = UI::Button.new(
        position: @bounding_rectangle.position.map {|v| v + 20},
        size: {200, 80},
        text: "btn",
        on_click: -> { p "Click!" }
      )
    end

    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      background = SF::RectangleShape.new(@bounding_rectangle)
      background.fill_color = Constants::COLOR::MENU::BACKGROUND

      target.draw(background, states)
      target.draw(@button, states)
    end

    def handle_event(event : SF::Event::MouseButtonPressed)
      if @bounding_rectangle.contains?(event.x, event.y)
        @button.handle_event(event)
      end
    end

    def handle_event(event)
    end
  end
end
```

We moved the button definition to initializer and we're caching it in an instance variable. In the `#draw`, we're rendering the instance variable, and we're defining a `#handle_event` method, which reacts to a `SF::Event::MouseButtonPressed` event. It then checks if the event is inside `@bounding_rectangle`, and if it is, we are passing the event to the `@button`. And the final piece:

```crystal
# src/lib/ui/button.cr class UI::Button

def handle_event(event : SF::Event::MouseButtonPressed)
  if @bounding_rectangle.contains?(event.x, event.y)
    @on_click.call
  end
end
```

We are defining a `#handle_event` on `Button`. If it contains the event, we call the `@on_click` function. That works just fine. But since our confirmation for working is something printed to the console, it's rather anticlimactic.

I have a better idea:

```crystal
# src/the_empire/bottom_menu.cr

class TheEmpire
  class BottomMenu
    include SF::Drawable

    def initialize(position, size)
      @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])
      @button1 = UI::Button.new(
        position: { @bounding_rectangle.position[0] + 20, @bounding_rectangle.position[1] + 20},
        size: {200, 80},
        text: "btn1",
        on_click: ->(button : UI::Button) {
          deactivate_all!
          activate(button)
        }
      )
      @button2 = UI::Button.new(
        position: { @bounding_rectangle.position[0] + 240, @bounding_rectangle.position[1] + 20},
        size: {200, 80},
        text: "btn2",
        on_click: ->(button : UI::Button) {
          deactivate_all!
          activate(button)
        }
      )
    end

    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      background = SF::RectangleShape.new(@bounding_rectangle)
      background.fill_color = Constants::COLOR::MENU::BACKGROUND

      target.draw(background, states)
      target.draw(@button1, states)
      target.draw(@button2, states)
    end

    def handle_event(event : SF::Event::MouseButtonPressed)
      if @bounding_rectangle.contains?(event.x, event.y)
        @button1.handle_event(event)
        @button2.handle_event(event)
      end
    end

    def handle_event(event)
    end

    private def deactivate_all!
      @button1.active = false
      @button2.active = false
    end

    private def activate(button : UI::Button)
      button.active = true
    end
  end
end
```

We updated the `BottomMenu` to have 2 buttons. Let's zoom in on a tricky part of this. We pass this to both buttons as action:

```crystal
on_click: ->(button : UI::Button) {
  deactivate_all!
  activate(button)
}
```

What I really want to achieve is this: If a button is clicked, set it as active, and set the other one as inactive. My initial idea for implementation was this:

```crystal
on_click: -> {
  @button1.active = true
  @button2.active = false
}
```

But that doesn't compile, and we get the following error:
```
Instance variable '@button1' was used before it was initialized in one of the 'initialize' methods, rendering it nilable
```

While initializing `@button1`, we are trying to create a `Proc`, that depends on `@button1`, and compiler doesn't like that.

The implementation I ended up with isn't exactly elegant, but it gets the job done. It also required a small update to the `UI::Button`:

```crystal
# src/lib/ui/button.cr

module UI
  class Button
    include SF::Drawable

    @bounding_rectangle : SF::IntRect
    @text : String
    @on_click : Proc(UI::Button, Bool)

    property active : Bool = false

    def initialize(position, size, text, on_click)
      @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])
      @text = text
      @on_click = on_click
    end

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

    def handle_event(event : SF::Event::MouseButtonPressed)
      if @bounding_rectangle.contains?(event.x, event.y)
        @on_click.call(self)
      end
    end
  end
end
```

The button will now call `@on_click` with `self`, and there is an `#active` boolean property, which changes the background color.

And that should produce a sufficient spectacle:

<video width="100%" src="/the_empire_blog/docs/assets/posts/7/spectacle.mp4" controls autoplay></video>

And that's it. In the next post, we will be introducing our first [modes of operation](8-modes-of-operation.html)!
