## Rendering more things in the world map

All right, so what happens if we want to render more stuff in the world map?

Our rendering pipeline currently consists of this in initializer:

```crystal
# src/the_empire/world_map.cr, class TheEmpire::WorldMap#initialize

@shape = SF::CircleShape.new(300)
@shape.fill_color = SF::Color::Black
```

And this:

```crystal
# src/the_empire/world_map.cr, class TheEmpire::WorldMap

def draw(target : SF::RenderTarget, states : SF::RenderStates)
  target.draw(@shape, states)
end
```

So what if we just keep adding stuff in the same manner? Let's try that:

```crystal
# src/the_empire/world_map.cr, class TheEmpire::WorldMap#initialize

@shape = SF::CircleShape.new(300)
@shape.fill_color = SF::Color::Black

@rectangle = SF::RectangleShape.new
@rectangle.size = SF.vector2f(200, 100)
@rectangle.outline_color = SF::Color::Red
@rectangle.outline_thickness = 5
@rectangle.position = {10, 20}
```

```crystal
# src/the_empire/world_map.cr, class TheEmpire::WorldMap

def draw(target : SF::RenderTarget, states : SF::RenderStates)
  target.draw(@shape, states)
  target.draw(@rectangle, states)
end
```

That obviously doesn't do what we want:

<video width="100%" src="/the_empire_blog/docs/assets/posts/5/multiple_objects.mp4" controls autoplay></video>

We have the usual black ball, and a new white/red rectangle. The black ball reacts to our inputs like it used to, while the rectangle just hangs there.
This is hardly surprising, we are applying all of the transformations to our black ball shape directly.

What we want, is all of the entities inside the world map reacting to the transformations in the same way.
The solution to this problem is...

## A view

[That one, specifically](https://oprypin.github.io/crsfml/api/SF/View.html). From the docs:

> SF::View defines a camera in the 2D scene. This is a very powerful concept: you can scroll, rotate or zoom the entire scene without altering the way that your drawable objects are drawn.

Well, then. Let's add one:

```crystal
# src/the_empire/world_map.cr, class TheEmpire::WorldMap#initialize

@view = SF::View.new(position, size)
```

We need to update out `#move` and `#scale` methods to update the `@view`:

```crystal
# src/the_empire/world_map.cr, class TheEmpire::WorldMap

def move(vector)
  @view.move(vector)
end

def scale(factor)
  @view.zoom(factor)
end
```

And the final thing: we need to tell the `target` to use the `@view` before rendering, and tell it to go back to the default one after rendering:

```crystal
# src/the_empire/world_map.cr, class TheEmpire::WorldMap

def draw(target : SF::RenderTarget, states : SF::RenderStates)
  target.view = @view

  target.draw(@shape, states)
  target.draw(@rectangle, states)

  target.view = target.default_view
end
```

Which gives us this:

<video width="100%" src="/the_empire_blog/docs/assets/posts/5/multiple_objects_with_view.mp4" controls autoplay></video>

Great success! Both objects now move when we move and zoom. In addition to that:
- shapes defiantly move in the reverse direction
- shapes seem distorted
- shapes are rendered in a different part of screen

## Bugfixing

Well, the first one is easy enough. We'll just reverse the <del>polarity of a neutron flow</del> direction:

```crystal
# src/the_empire/world_map.cr, class TheEmpire::WorldMap

def move(vector)
  reversed_vector = vector.map { |value| -value }
  @view.move(reversed_vector)
end
```

If you are told to move in one direction, move to a reverse one. Boom, fixed:

<video width="100%" src="/the_empire_blog/docs/assets/posts/5/multiple_objects_with_view_2.mp4" controls autoplay></video>

That's the first problem gone.

Notice how I grabbed the view by the top-left corner of the rectangle, but it seems to slip away as I move. The shapes move at a faster rate than the mouse in all directions.
That's a very important clue: view thinks I am moving in the window by more than I actually am.

If we check [the docs](https://oprypin.github.io/crsfml/api/SF/View.html)!, the solution to the problem is stated quite clearly (emphasis mine):

> A view is composed of a source rectangle, which defines what part of the 2D scene is shown, and a **target viewport, which defines where the contents of the source rectangle will be displayed on the render target (window or texture)**.

> If the source rectangle doesn't have the same size as the viewport, its **contents will be stretched to fit in**.

Right, we need to set the viewport. To do that, we need to know where the contents will be rendered, as a percentage of the full view. Currently, we are initializing `TheEmpire::WorldMap` with position and size, so we only know the size of the `WorldMap`. We need to know the size of full window as well, and to do that, I'll additionally pass the `window` object to it:

```crystal
@ src/the_empire/world_map.cr, class TheEmpire::WorldMap

def initialize(position, size, @window)
  @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])

  @view = SF::View.new(position, size)
  @view.viewport = SF::FloatRect.new(
    0.0,
    0.0,
    (size[0] / @window.size.x).to_f32,
    (size[1] / @window.size.y).to_f32
  )

  ...
end
```

This view represents viewport, which starts in the top-left corner (point 0, 0) of the screen, and is rendered in the left `size[0] / @window.size.x` percent of the width of the window, and `size[1] / @window.size.y` percent of the height of the window.

I'll update `TheEmpire#initialize` accordingly in the background. And there it is:

<video width="100%" src="/the_empire_blog/docs/assets/posts/5/multiple_objects_done.mp4" controls autoplay></video>

The shapes are moving correctly, and they are no longer stretched. The only remaining issue is the fact, that objects are initially rendered in a different place.
But that's not issue at all. It looks like due to the view, `WorldMap` considers the point `(0, 0)` to be in the middle of the `WorldMap`, and I quite like that. I'll leave it like this.

I don't like how we updated the `move` logic to do the reverse, so I'll update the code to have the `handle_event(event : SF::Event::KeyPressed)` and `handle_event(event : SF::Event::MouseMoved)` produce the correct values in the first place.

We end up with code like this:

```crystal
# src/the_empire/world_map.cr

class TheEmpire
  class WorldMap
    include SF::Drawable

    MOVING_SPEED = 10

    @window : SF::RenderWindow

    property moving_up_speed = 0, moving_left_speed = 0

    def initialize(position, size, @window)
      @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])

      @view = SF::View.new(position, size)
      @view.viewport = SF::FloatRect.new(
        0.0,
        0.0,
        (size[0] / @window.size.x).to_f32,
        (size[1] / @window.size.y).to_f32
      )

      @shape = SF::CircleShape.new(300)
      @shape.fill_color = SF::Color::Black

      @rectangle = SF::RectangleShape.new
      @rectangle.size = SF.vector2f(200, 100)
      @rectangle.outline_color = SF::Color::Red
      @rectangle.outline_thickness = 5
      @rectangle.position = {10, 20}

      @moving_around = false
      @mouse_button_initial_x = 0
      @mouse_button_initial_y = 0
    end

    def handle_event(event : SF::Event::KeyPressed)
      case event.code
      when SF::Keyboard::Key::W then self.moving_up_speed = MOVING_SPEED
      when SF::Keyboard::Key::S then self.moving_up_speed = -MOVING_SPEED
      when SF::Keyboard::Key::A then self.moving_left_speed = MOVING_SPEED
      when SF::Keyboard::Key::D then self.moving_left_speed = -MOVING_SPEED
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
        x_delta = @mouse_button_initial_x - event.x
        y_delta = @mouse_button_initial_y - event.y

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
      @view.move(vector)
    end

    def scale(factor)
      @view.zoom(factor)
    end

    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      target.view = @view

      target.draw(@shape, states)
      target.draw(@rectangle, states)

      target.view = target.default_view
    end
  end
end
```

And we're done! Now, that we have more-or-less functioning rendering pipeline, I'd like to start implementing some actual drawing features.
First step to do that is [implementing a button](6-implementing-a-button.html)!
