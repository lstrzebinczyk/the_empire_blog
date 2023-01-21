Buckle up, this one is long and heavy on architecture.

## What are modes?

We want to be able to interact with the world map in multiple ways. Think of a paint program: you have a tool to draw straight lines, or pencil, or coloring tool. You interact with the canvas in the same way, but depending on the tool, the result on a canvas is different.

For this app, I've come to think of these "tools" as modes. Modes are mediators between user interaction and the world map. We want the currently active mode to process events, and interact with the world map according to their logic.

For now, we will be thinking about introducing first 2 modes:
- `MoveMode`, allowing to move the map around and basic introspection
- `POIMode`, allowing management of points of interest

Fortunetely, we already have the entire behavior of `MoveMode` implemented, we just need to encapsulate it in a class.

## Modes architecture

The classes will be called `TheEmpire::Mode::MoveMode` and `TheEmpire::Mode::POIMode`. We'll begin by thinking how they should be used in the app:
1. We want them to accept events instead of `WorldMap`, so we need to initialize it in `TheEmpire`, since this is where these events happen
1. We want them to update `WorldMap`, so we'll pass that in an initializer.
1. One of the modes will always be active
1. Bottom menu should render buttons for all available modes, with clear indication which one is currently active
1. Modes should be able to render to the world map

So let's see how we can extend `TheEmpire` to get all of these requirements done.

We initialize the modes in `TheEmpire#initialize`:

```crystal
# src/the_empire.cr, class TheEmpire#initialize
# below initializing of @world_map

@move_mode = TheEmpire::Mode::MoveMode.new(@world_map)
@poi_mode = TheEmpire::Mode::POIMode.new(@world_map)

@modes = [
  @move_mode,
  @poi_mode
]

@active_mode = @move_mode
```

We'll pass the modes data to the `BottomMenu`:

```crystal
# src/the_empire.cr, class TheEmpire#initialize

@bottom_menu = TheEmpire::BottomMenu.new(
  position: {0, @window_height - BOTTOM_MENU_HEIGHT},
  size: {@window_width - RIGHT_MENU_WIDTH, BOTTOM_MENU_HEIGHT},
  modes: @modes,
  active_mode: @active_mode
)
```

And finally, we'll update the `#handle_event` to use `@active_mode` instead of `@world_map`, as well as render the contents of `@active_mode`:

```crystal
# src/the_empire.cr, class TheEmpire#initialize

def handle_event(event)
  @active_mode.handle_event(event) # <- this was @world_map
  @bottom_menu.handle_event(event)
  @right_menu.handle_event(event)
end

def render
  @window.clear(SF::Color::White)

  @window.draw(@world_map)
  @window.draw(@active_mode) # <- this is new
  @window.draw(@bottom_menu)
  @window.draw(@right_menu)

  @window.display
end
```

Perfect. This is how we want these classess to interact with the app. Let's fix all the ways in which we broke the app. I'll be showing the `MoveMode`, but `POIMode` will get identical treatment. So:

```bash
In src/the_empire.cr:28:44

 28 | @move_mode = TheEmpire::Mode::MoveMode.new(@world_map)
                                             ^--
Error: wrong number of arguments for 'TheEmpire::Mode::MoveMode.new' (given 1, expected 0)
```

Of course, the modes must accept world map as argment:

```crystal
# src/the_empire/mode/move_mode.cr

class TheEmpire
  module Mode
    class MoveMode
      def initialize(@world_map : TheEmpire::WorldMap)
      end
    end
  end
end
```

Next!

```bash
In src/the_empire.cr:38:42

 38 | @bottom_menu = TheEmpire::BottomMenu.new(
                                           ^--
Error: no overload matches 'TheEmpire::BottomMenu.new', position: Tuple(Int32, Int32), size: Tuple(Int32, Int32), modes: Array(TheEmpire::Mode::MoveMode | TheEmpire::Mode::POIMode), active_mode: TheEmpire::Mode::MoveMode

Overloads are:
 - TheEmpire::BottomMenu.new(position, size)
```

This one is a bit more interesting. Of course, we passed additional arguments to `TheEmpire::BottomMenu`, and it doesn't know how to handle them. Look at the arguments it tells us it got:

```crystal
modes: Array(TheEmpire::Mode::MoveMode | TheEmpire::Mode::POIMode), active_mode: TheEmpire::Mode::MoveMode
```

It tells us that `modes` is an array of either `MoveMode` or `POIMode`, and `active_mode` is an `MoveMode`. That is true, of course, but it is too specific. We need to have an idea that all of these are **some** modes.
We'll introduce a parent class for these modes, to enforce unity:

```crystal
# src/the_empire/mode/base_mode.cr

class TheEmpire
  module Mode
    abstract class BaseMode
      def initialize(@world_map : TheEmpire::WorldMap)
      end
    end
  end
end

```

Make sure that parent class is used:

```crystal
# src/the_empire/mode/move_mode.cr

class TheEmpire
  module Mode
    class MoveMode < TheEmpire::Mode::BaseMode
      def initialize(world_map : TheEmpire::WorldMap)
        super(world_map)
      end
    end
  end
end
```

So now we can extend the `BottomMenu`. We'll simply accept these modes as params and assign them to instance variables for now:

```crystal
# src/the_empire/bottom_menu.cr, class TheEmpire::BottomMenu

@modes : Array(TheEmpire::Mode::BaseMode)
@active_mode : TheEmpire::Mode::BaseMode

def initialize(position, size, @modes, @active_mode)
  ...
end
```

Next!

```bash
Showing last frame. Use --error-trace for full trace.

In src/the_empire.cr:67:18

 67 | @active_mode.handle_event(event)
                   ^-----------
Error: undefined method 'handle_event' for TheEmpire::Mode::MoveMode
```

Of course, modes must handle events:

```crystal
# src/the_empire/modes/move_mode.cr

class TheEmpire
  module Mode
    class MoveMode < TheEmpire::Mode::BaseMode
      def initialize(world_map : TheEmpire::WorldMap)
        super(world_map)
      end

      def handle_event(event)
      end
    end
  end
end
```

Next!

```bash
Showing last frame. Use --error-trace for full trace.

In src/the_empire.cr:80:13

 80 | @window.draw(@active_mode)
              ^---
Error: no overload matches 'SF::RenderWindow#draw' with type TheEmpire::Mode::MoveMode
```

Naturally, we want the modes to be drawable. We'll include the `SF::Drawable` into the parent class:

```crystal
# src/the_empire/mode/base_mode.cr

class TheEmpire
  module Mode
    abstract class BaseMode
      include SF::Drawable

      def initialize(@world_map : TheEmpire::WorldMap)
      end
    end
  end
end
```

And add empty `#draw` method to the mode:

```crystal
# src/the_empire/modes/move_mode.cr

class TheEmpire
  module Mode
    class MoveMode < TheEmpire::Mode::BaseMode
      def initialize(world_map : TheEmpire::WorldMap)
        super(world_map)
      end

      def handle_event(event)
      end

      def draw(target : SF::RenderTarget, states : SF::RenderStates)
      end
    end
  end
end
```

And that compiles correctly! Now, before we get to the exciting part of implementing the actual mode behavior, we need to think a bit about...

## Mode buttons

We need to be able to freely pick which mode we are using, and that presents a little bit of a challange.
We are going to solve this challange with the best `react.js` taught us: data down, actions up.

### Data down

We pass `@modes` to the initializer of `BottomMenu`, so we can use them to render the corresponding buttons. I'm going to do what is called a pro-coder move:

```crystal
# src/the_empire/mode/base_mode.cr

class TheEmpire
  module Mode
    abstract class BaseMode
      include SF::Drawable

      getter button_text : String

      def initialize(@world_map : TheEmpire::WorldMap)
        @button_text = self.class.to_s.split("::").last.gsub("Mode", "")
      end
    end
  end
end
```

This little trick will calculate a variable called `@button_text` for the modes, based on the class name.
It will return `Move` for `MoveMode` and `POI` for `POIMode`, easy.

In `BottomMenu` we need to create the buttons based on passed `@modes`:

```crystal
# src/the_empire/bottom_menu.cr, class TheEmpire::BottomMenu

# We are now going to have an array of buttons
@buttons : Array(UI::Button)

def initialize(position, size, @modes, @active_mode)
  @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])

  # easy peasy
  @buttons = @modes.map_with_index do |mode, index|
    UI::Button.new(
      position: { @bounding_rectangle.position[0] + 20 + index * 220, @bounding_rectangle.position[1] + 20},
      size: {200, 80},
      text: mode.button_text,
      active: mode == @active_mode,
      on_click: ->(button : UI::Button) {
        deactivate_all!
        activate(button)
      }
    )
  end
end
```

The rest of the class is easily updated to perform all actions on `@buttons.each`. Also I extended the `UI::Button` to accept `active` as param too.
That covers the "data down" part:

![Correct buttons](/the_empire_blog/docs/assets/posts/8/correct_buttons.png)

Awesome, so the easy part is done. Our buttons are now rendered based on the `@modes`.

### Actions up

The concept of `Actions up` means, that if an action happens in child components (like `BottomMenu`) jurisdiction, it should tell parent component (`TheEmpire`) about it, and let it handle the outcome.
In case at hand, if a button inside `BottomMenu` is clicked, that means `TheEmpire` must update which mode is the active one.

The thing is, that we are going to need a way to send any custom information that something happened, alongside any custom data, from any child component back to `TheEmpire`.

We're gonna get that done with events. Not `CrSFML` events, we will add new ones, specifically for that purpose:

```crystal
# src/the_empire/event.cr

class TheEmpire
  abstract struct Event
    struct ChangeModeEvent < Event
      getter mode

      def initialize(@mode : TheEmpire::Mode::BaseMode)
      end
    end
  end
end
```

We have a `TheEmpire::Event::ChangeModeEvent` struct, with a `mode` value. `TheEmpire` must be ready to handle it:

```crystal
# src/the_empire.cr, class TheEmpire

def handle_page_event(event : TheEmpire::Event)
  case event
  when TheEmpire::Event::ChangeModeEvent
    @active_mode = event.mode
  end

  nil # we want to return nil, just to keep types clean
end
```

If we call that method with an instance of `TheEmpire::Event::ChangeModeEvent`, we'll change the `@active_mode` to one we received in the event.

Next step, is passing that new method to `BottomMenu`:

```crystal
# src/the_empire.cr, class TheEmpire#initialize

@bottom_menu = TheEmpire::BottomMenu.new(
  position: {0, @window_height - BOTTOM_MENU_HEIGHT},
  size: {@window_width - RIGHT_MENU_WIDTH, BOTTOM_MENU_HEIGHT},
  modes: @modes,
  active_mode: @active_mode,
  omit_event: ->handle_page_event(TheEmpire::Event) # <- this is new
)
```

This wonderful piece of coding trickery means we are sending `#handle_page_event` method as a parameter to the `BottomMenu` class.
Now we can omit the event from `BottomMenu`:

```crystal
# src/the_empire/bottom_menu.cr, class TheEmpire::BottomMenu

@omit_event : Proc(TheEmpire::Event, Nil)

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
```

When we click on any of these buttons, we create a `TheEmpire::Event::ChangeModeEvent` and call the received `@omit_event`, which then processess the event inside `TheEmpire`.
Finally, we must update the `@handle_page_event` with this:

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
  end

  nil # we want to return nil, just to keep types clean
end
```

When processing the event, we need to rebuild the `@bottom_menu` from scratch after changing the `@active_mode`. This is because `BottomMenu` decides which button is active based on `@active_mode`, and after we've changed it, that change isn't propagated to `BottomMenu`. So we create it again, it gets the correct `@active_mode`, and indicates the program state correctly.

Long and difficult road led here, but alas, victory:

<video width="100%" src="/the_empire_blog/docs/assets/posts/8/working_buttons.mp4" controls autoplay></video>

While that might look exactly like it looked at the beginning of this post, we have developed important architecture! Actually, it looks like even less is working than used to, since currently we can't even move things.
But that's easy, a cherry on top.

## Modes behavior

I want all modes to move around with `AWSD` buttons, so we'll handle that event in the parent class:

```crystal
# src/the_empire/mode/base_mode.cr

class TheEmpire
  module Mode
    abstract class BaseMode
      include SF::Drawable

      MOVING_SPEED = 10

      getter button_text : String

      def initialize(@world_map : TheEmpire::WorldMap)
        @button_text = self.class.to_s.split("::").last.gsub("Mode", "")
      end

      def handle_event(event)
        case event
        when SF::Event::KeyPressed
          case event.code
          when SF::Keyboard::Key::W then @world_map.moving_up_speed = MOVING_SPEED
          when SF::Keyboard::Key::S then @world_map.moving_up_speed = -MOVING_SPEED
          when SF::Keyboard::Key::A then @world_map.moving_left_speed = MOVING_SPEED
          when SF::Keyboard::Key::D then @world_map.moving_left_speed = -MOVING_SPEED
          end
        when SF::Event::KeyReleased
          case event.code
          when SF::Keyboard::Key::W then @world_map.moving_up_speed = 0
          when SF::Keyboard::Key::S then @world_map.moving_up_speed = 0
          when SF::Keyboard::Key::A then @world_map.moving_left_speed = 0
          when SF::Keyboard::Key::D then @world_map.moving_left_speed = 0
          end
        end
      end
    end
  end
end
```

`POIMode` will just call the parent action:

```crystal
# src/the_empire/mode/poi_mode.cr

class TheEmpire
  module Mode
    class POIMode < TheEmpire::Mode::BaseMode
      def initialize(world_map : TheEmpire::WorldMap)
        super(world_map)
      end

      def handle_event(event)
        super(event)
      end

      def draw(target : SF::RenderTarget, states : SF::RenderStates)
      end
    end
  end
end
```

And `MoveMode` will get the ability to drag-and-drop and scroll:

```crystal
# src/the_empire/mode/move_mode.cr

class TheEmpire
  module Mode
    class MoveMode < TheEmpire::Mode::BaseMode
      def initialize(world_map : TheEmpire::WorldMap)
        super(world_map)

        @moving_around = false
        @mouse_button_initial_x = 0
        @mouse_button_initial_y = 0
      end

      def handle_event(event)
        super(event)

        case event
        when SF::Event::MouseButtonPressed
          if @world_map.bounding_rectangle.contains?(event.x, event.y)
            @moving_around = true
            @mouse_button_initial_x = event.x
            @mouse_button_initial_y = event.y
          end
        when SF::Event::MouseButtonReleased
          @moving_around = false
        when SF::Event::MouseMoved
          if @moving_around
            x_delta = @mouse_button_initial_x - event.x
            y_delta = @mouse_button_initial_y - event.y

            @world_map.move({x_delta, y_delta})

            @mouse_button_initial_x = event.x
            @mouse_button_initial_y = event.y
          end
        when SF::Event::MouseWheelMoved
          if event.delta > 0
            @world_map.scale(1.25)
          else
            @world_map.scale(0.8)
          end
        end
      end

      def draw(target : SF::RenderTarget, states : SF::RenderStates)
      end
    end
  end
end
```

And this is the end game:

<video width="100%" src="/the_empire_blog/docs/assets/posts/8/actually_separate_modes.mp4" controls autoplay></video>

`MoveMode` supports `AWSD` and mouse actions, and `POIMode` supports `AWSD`, but doesn't react to mouse actions (trust me, I guess).

And that's a big win! We will add actual drawing behavior to `POIMode` soon enough.
For now, we need to extend our UI capabilities a little bit, starting with [right menu, observable and text](9-right-menu-observable-and-text.html)!
