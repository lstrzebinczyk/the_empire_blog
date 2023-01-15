Let's try and move that circle around a little bit. We want to:
- use the AWSD to move it around
- drag and drop it with mouse
- zoom in/out with the mouse scroll.

I'll be following the tutorial on [transforming entities](https://oprypin.github.io/crsfml/tutorials/graphics/transform.html) and [managing events](https://oprypin.github.io/crsfml/tutorials/window/events.html).

If we want to move our circle around, let's cache it in the initializer, and render from the instance variable:

```crystal
# src/the_empire.cr

class TheEmpire
  def initialize
    ...

    @shape = SF::CircleShape.new(300)
    @shape.fill_color = SF::Color::Black
  end

  ...

  def render
    @window.clear(SF::Color::White)
    @window.draw(@shape)
    @window.display
  end
end
```

Since we already have the system of handling events, we just need to handle them:

```crystal
# src/the_empire.cr, class TheEmpire

def handle_event(event : SF::Event::KeyPressed)
  case event.code
  when SF::Keyboard::Key::W then @shape.move(0, MOVING_SPEED)
  when SF::Keyboard::Key::S then @shape.move(0, -MOVING_SPEED)
  when SF::Keyboard::Key::A then @shape.move(MOVING_SPEED, 0)
  when SF::Keyboard::Key::D then @shape.move(-MOVING_SPEED, 0)
  end
end
```

If a `SF::Event::KeyPressed` event is received, for the key codes of A, W, S and D, `@shape.move` around. Let's see:

<video width="100%" src="/the_empire_blog/docs/assets/posts/3/janky_moves.mp4" controls autoplay></video>

Well, that works, but holy smokes, it's SO JANKY... Looking more carefully at the tutorial finds me this piece:

> Sometimes, people try to react to KeyPressed events directly to implement smooth movement. Doing so will not produce the expected effect, because when you hold a key you only get a few events (remember, the repeat delay). To achieve smooth movement with events, you must use a boolean that you set on KeyPressed and clear on KeyReleased; you can then move (independently of events) as long as the boolean is set.

Allrighty then, let's define the additional values:

```crystal
# src/the_empire.cr, class TheEmpire

property moving_up_speed = 0, moving_left_speed = 0
```

And we're gonna have to update howwe deal with `SF::Event::KeyPressed`:

```crystal
# src/the_empire.cr, class TheEmpire

def handle_event(event : SF::Event::KeyPressed)
  case event.code
  when SF::Keyboard::Key::W then self.moving_up_speed = -MOVING_SPEED
  when SF::Keyboard::Key::S then self.moving_up_speed = MOVING_SPEED
  when SF::Keyboard::Key::A then self.moving_left_speed = -MOVING_SPEED
  when SF::Keyboard::Key::D then self.moving_left_speed = MOVING_SPEED
  end
end
```

This will set the status as moving in the given direction, when we press a key. Now, to stop the movement once we release:

```crystal
# src/the_empire.cr, class TheEmpire

def handle_event(event : SF::Event::KeyReleased)
  case event.code
  when SF::Keyboard::Key::W then self.moving_up_speed = 0
  when SF::Keyboard::Key::S then self.moving_up_speed = 0
  when SF::Keyboard::Key::A then self.moving_left_speed = 0
  when SF::Keyboard::Key::D then self.moving_left_speed = 0
  end
end
```

This, by itself, doesn't do anything. We need to introduce additional element into the main loop: updating;

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

First we handle any user events, then we update the world, and finally render. For now, the update is very simple:

```crystal
# src/the_empire.cr, class TheEmpire

def update
  if moving_up_speed != 0 || moving_left_speed != 0
    move({moving_left_speed, moving_up_speed})
  end
end

def move(vector)
  @shape.move(vector)
end
```

Et voila:

<video width="100%" src="/the_empire_blog/docs/assets/posts/3/smooth_moves.mp4" controls autoplay></video>

I am aware it doesn't immedietely look all that better, but that is because of poor quality of the recorded clip. In actual app, it's silky smooth.

Second thing I want to do is to drag and drop the ball. There are 3 events, which I'm interested in for this: `SF::Event::MouseButtonPressed`, `SF::Event::MouseButtonReleased` and `SF::Event::MouseMoved`.
This will happen in 3 stages:
- on `SF::Event::MouseButtonPressed`, we will mark the world as being dragged
- on `SF::Event::MouseMoved`, if the world is being dragged, update the `@shape` according to the dragging
- on `SF::Event::MouseButtonReleased`, we will mark the world as no longer being dragged

This would have been trivial if `SF::Event::MouseMoved` had deltas (that is, gave us information about how much movement happened since the last one), but it doesn't. We will have to calculate it ourselves:

```crystal
# src/the_empire.cr, class TheEmpire

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
    x_delta = event.x - @mouse_button_initial_x
    y_delta = event.y - @mouse_button_initial_y

    move({x_delta, y_delta})

    @mouse_button_initial_x = event.x
    @mouse_button_initial_y = event.y
  end
end
```

Which gives us this beauty:

<video width="100%" src="/the_empire_blog/docs/assets/posts/3/drag_and_drop.mp4" controls autoplay></video>

This leaves us with zooming:

```crystal
# src/the_empire.cr, class TheEmpire

def handle_event(event : SF::Event::MouseWheelMoved)
  if event.delta > 0
    scale(1.25)
  else
    scale(0.8)
  end
end

def scale(factor)
  @shape.scale(factor, factor)
end
```

When we scroll up, delta is positive, and we want to make things 25% bigger. When we scroll down, delta is negative, and we want to make things 20% smaller.
This way, if we scroll up, scrolling down will set us back on the value we were before. No chance we'll end up on any weird values.

Easy enough:

<video width="100%" src="/the_empire_blog/docs/assets/posts/3/scaling.mp4" controls autoplay></video>

And there it is. We can now move our black ball around and zoom it. In full, the code is currently like this:

```crystal
require "crsfml"

class TheEmpire
  MOVING_SPEED = 10

  property moving_up_speed = 0, moving_left_speed = 0

  def initialize
    @window_width = 1920
    @window_height = 1080

    @window = SF::RenderWindow.new(SF::VideoMode.new(@window_width, @window_height), "My window")
    # If I don't do that, it actually renders ~20000 FPS, lol
    @window.framerate_limit = 60
    # I have 3 screens, and if I don't set it, it renders in my left-most screen.
    # I would appreciate the option to define main screen, actually.
    @window.position = SF.vector2(6500, 1800)

    @shape = SF::CircleShape.new(300)
    @shape.fill_color = SF::Color::Black

    @moving_around = false
    @mouse_button_initial_x = 0
    @mouse_button_initial_y = 0
  end

  def running?
    @window.open?
  end

  def handle_events
    while event = @window.poll_event
      handle_event(event)
    end
  end

  # Handle the close event specifically
  def handle_event(event : SF::Event::Closed)
    @window.close
  end

  def handle_event(event : SF::Event::KeyPressed)
    case event.code
    when SF::Keyboard::Key::W then self.moving_up_speed = -MOVING_SPEED
    when SF::Keyboard::Key::S then self.moving_up_speed = MOVING_SPEED
    when SF::Keyboard::Key::A then self.moving_left_speed = -MOVING_SPEED
    when SF::Keyboard::Key::D then self.moving_left_speed = MOVING_SPEED
    end
  end

  def handle_event(event : SF::Event::KeyReleased)
    case event.code
    when SF::Keyboard::Key::W then self.moving_up_speed = 0
    when SF::Keyboard::Key::S then self.moving_up_speed = 0
    when SF::Keyboard::Key::A then self.moving_left_speed = 0
    when SF::Keyboard::Key::D then self.moving_left_speed = 0
    end
  end

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
      x_delta = event.x - @mouse_button_initial_x
      y_delta = event.y - @mouse_button_initial_y

      move({x_delta, y_delta})

      @mouse_button_initial_x = event.x
      @mouse_button_initial_y = event.y
    end
  end

  def handle_event(event : SF::Event::MouseWheelMoved)
    if event.delta > 0
      scale(1.25)
    else
      scale(0.8)
    end
  end

  # Ignore any other event
  def handle_event(event)
  end

  def update
    if moving_up_speed != 0 || moving_left_speed != 0
      move({moving_left_speed, moving_up_speed})
    end
  end

  def move(vector)
    @shape.move(vector)
  end

  def scale(factor)
    @shape.scale(factor, factor)
  end

  def render
    @window.clear(SF::Color::White)
    @window.draw(@shape)
    @window.display
  end
end
```

So now, that the basics are done, I would like to start thinking about the [UI for map painting app](4-initial-ui.html)!
