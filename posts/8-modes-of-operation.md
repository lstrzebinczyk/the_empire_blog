Buckle up, this one is long and heavy on architecture.

## What are modes?

We want to be able to interact with the world map in many ways. Think of a paint program: you have a tool to draw straight lines, or a pencil, or color filling tool. You click on the canvas all the same, but what happens depends on the active tool. The tools are mediators between user interaction and the canvas.

For this app, I've come to think of these "tools" as modes. Instead of letting the `WorldMap` to process events, we want the active mode to do so, and let it decide how it will update the `WorldMap`.

For now, we will be thinking about introducing first 2 modes:
- `MoveMode`, allowing to move the map around and basic introspection
- `POIMode`, allowing management of points of interest

Fortunetely, we already have the entire behavior of `MoveMode` implemented. We "just" need to encapsulate it in a class and insert it into the workflow.

## Modes architecture

The classes will be called `TheEmpire::Mode::MoveMode` and `TheEmpire::Mode::POIMode`. We'll begin by thinking how they should be used in the app:
1. They must process events instead of `WorldMap`
1. They should be able to update `WorldMap`
1. One of the modes must always be active
1. Bottom menu should render buttons for all available modes, with indication which one is active
1. Modes should be able to render to the world map

So let's see how we can extend `TheEmpire` to get all of these requirements done. As I often do, I'll start by establishing how I want the new classess to be used, and then I'll implement them to fit these expectations.

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

Notice they receive the `@world_map` in an argument. That's how they will be able to update it. We'll pass the modes data to the `BottomMenu`. That's how it will be able to render buttons for each mode:

```crystal
# src/the_empire.cr, class TheEmpire#initialize

@bottom_menu = TheEmpire::BottomMenu.new(
  position: {0, @window_height - BOTTOM_MENU_HEIGHT},
  size: {@window_width - RIGHT_MENU_WIDTH, BOTTOM_MENU_HEIGHT},
  modes: @modes,
  active_mode: @active_mode
)
```

And finally, we'll replace `@world_map` with `@active_mode` in `TheEmpire#handle_event` and render the contents of `@active_mode` in addition to other renders:

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

Perfect. The initial implementation of modes will be fairly simple:

```crystal
# src/the_empire/modes/move_mode.cr

class TheEmpire
  module Mode
    class MoveMode
      include SF::Drawable

      def initialize(@world_map : TheEmpire::WorldMap)
      end

      def handle_event(event)
      end

      def draw(target : SF::RenderTarget, states : SF::RenderStates)
      end
    end
  end
end
```

Moving on, compiler gives us this error about `BottomMenu`:

```bash
In src/the_empire.cr:38:42

 38 | @bottom_menu = TheEmpire::BottomMenu.new(
                                           ^--
Error: no overload matches 'TheEmpire::BottomMenu.new', position: Tuple(Int32, Int32), size: Tuple(Int32, Int32), modes: Array(TheEmpire::Mode::MoveMode | TheEmpire::Mode::POIMode), active_mode: TheEmpire::Mode::MoveMode

Overloads are:
 - TheEmpire::BottomMenu.new(position, size)
```

Of course, we passed additional arguments to `TheEmpire::BottomMenu`, and it doesn't know how to handle them. Look at the new arguments:

```crystal
modes: Array(TheEmpire::Mode::MoveMode | TheEmpire::Mode::POIMode), active_mode: TheEmpire::Mode::MoveMode
```

It tells us that `modes` is an array of either `MoveMode` or `POIMode`, and `active_mode` is an `MoveMode`. While this is true, it's way too specific. We need to work with an idea of **a** mode. All of the modes should be recognized as specialized instances of the same tool.

We'll introduce a parent class and inherit:

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

Make sure that parent class is used:

```crystal
# src/the_empire/mode/move_mode.cr

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

Now we can extend the `BottomMenu` based on it. For now, we'll accept the new arguments and assign them to instance variables:

```crystal
# src/the_empire/bottom_menu.cr, class TheEmpire::BottomMenu

@modes : Array(TheEmpire::Mode::BaseMode)
@active_mode : TheEmpire::Mode::BaseMode

def initialize(position, size, @modes, @active_mode)
  ...
end
```

And that compiles correctly! Now, before we get to the exciting part of implementing the actual mode behavior, we need to think a bit about...

## Mode buttons

We need to be able to freely pick an active mode. That presents a little bit of a challange. `BottomMenu` can render the buttons, because it got modes in initializer. That's easy.
When one of these buttons is clicked, `BottomMenu` must somehow be able to tell `TheEmpire`, that we want to use a different mode.

We'll solve this with the best `react.js` taught us: data down, actions up.

### Data down

That's the mentioned easy part. We pass `@modes` to the initializer of `BottomMenu`, so we can use them to render the corresponding buttons. I'm going to start this with what is called a pro-coder move:

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

This little trick implements a `#button_text` method for all modes, based on the class name.
It will return `Move` for `MoveMode` and `POI` for `POIMode`, easy.

In `BottomMenu` we can now create the buttons based on passed `@modes` and `@active_mode`:

```crystal
# src/the_empire/bottom_menu.cr, class TheEmpire::BottomMenu

@buttons : Array(UI::Button)

def initialize(position, size, @modes, @active_mode)
  @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])

  @buttons = @modes.map_with_index do |mode, index|
    UI::Button.new(
      # each button will be 220 pixels to the right from where the previous started
      position: { @bounding_rectangle.position[0] + 20 + index * 220, @bounding_rectangle.position[1] + 20},
      size: {200, 80},
      # text on the button is taken from the mode
      text: mode.button_text,
      # button is active if the mode is currently active
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

We have 2 buttons based on available modes, and the `Move` mode is currently active. Awesome. Moving to...

### Actions up

The concept of `Actions up` means, that child component (like `BottomMenu`) should be able to tell parent component (`TheEmpire`) that something happened and let it (the parent component) handle the outcome.
In this case, if a button in `BottomMenu` is clicked, `TheEmpire` must update `@active_mode`.

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

We have a `TheEmpire::Event::ChangeModeEvent` struct, which represents the information, that active mode must change. `TheEmpire` must be ready to handle it:

```crystal
# src/the_empire.cr, class TheEmpire

def handle_page_event(event : TheEmpire::Event)
  case event
  when TheEmpire::Event::ChangeModeEvent
    @active_mode = event.mode
  end

  nil # this helps to keep the `Proc` types below simple
end
```

`#handle_page_event` is now how `TheEmpire` is allowed to process `TheEmpire::Event` events coming from any child components.

Next step, we pass that new method to `BottomMenu`:

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

With this wonderful piece of coding trickery, we can turn `#handle_page_event` method into a `Proc` and pass it into `BottomMenu` under the name of `omit_event`.
We can now do this:

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

When we click on any of these buttons, we create a `TheEmpire::Event::ChangeModeEvent` and call the `@omit_event` `Proc`, which then processess the event inside `TheEmpire`.
Finally, we must update the `@handle_page_event` with this:

```crystal
# src/the_empire.cr, class TheEmpire

def handle_page_event(event : TheEmpire::Event)
  case event
  when TheEmpire::Event::ChangeModeEvent
    @active_mode = event.mode

    # this part is new
    @bottom_menu = TheEmpire::BottomMenu.new(
      position: {0, @window_height - BOTTOM_MENU_HEIGHT},
      size: {@window_width - RIGHT_MENU_WIDTH, BOTTOM_MENU_HEIGHT},
      modes: @modes,
      active_mode: @active_mode,
      omit_event: ->handle_page_event(TheEmpire::Event)
    )
  end

  nil
end
```

When processing the event, we change `@active_mode`, but that doesn't automatically propagate to `BottomMenu`. If we want the `BottomMenu` to render based on the new `@active_mode`, we need to initialize it again with updated data.

Long and difficult road led here, but alas, victory:

<video width="100%" src="/the_empire_blog/docs/assets/posts/8/working_buttons.mp4" controls autoplay></video>

While that looks exactly like what we had at the beginning, we have developed important architecture! Actually, even less is working than before, since we can't move things.
But that's easy, a cherry on top. Let's implement the...

## Modes behavior

I want all modes to move around with `AWSD` buttons, so we'll handle that in the parent class:

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

`MoveMode` will call the parent, plus get the ability to drag-and-drop and scroll:

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
        super(event) # <- This calls the `#handle_event` from parent class

        # this below handles drag-and-drop and scroll
        # it was just moved here from `WorldMap`
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

All of the event handling was removed from the `WorldMap`, as it was moved to `MoveMode`. `POIMode` will just call the parent action.

Grand finale, we end up with this:

<video width="100%" src="/the_empire_blog/docs/assets/posts/8/actually_separate_modes.mp4" controls autoplay></video>

`MoveMode` supports `AWSD` and mouse actions, and `POIMode` supports `AWSD`, but doesn't react to mouse actions (trust me, I guess).

Big win! We will add actual drawing behavior to `POIMode` soon enough.

For now, we need to extend our UI capabilities a little bit, starting with [a basic ui framework](9-a-basic-ui-framework.html)!
