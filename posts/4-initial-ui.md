At this point I am quite confident I have all of the tools to do what I want. So let's explore a little bit what that is.

When I say, that I want to simulate the economy, what I mean is: I want to define where specific goods are created, where these goods are required and used, and I want to see how they would travel, from the place of their creation, to where they are used. Since we are talking about Warhammer, the easiest and earliest example could be: wheat, cheese or rye are created at farms, but there is a market for all of these goods in cities. So a merchant would try and buy them cheaply at farms and transport them to the cities. Or maybe the farms would be taxed.
I would love to simulate this per-person, but for now we are aiming at simulating per city (or a town, or a farm).

So, when I say, I want to draw a map, I mean: I want to define where are the cities (or towns, or farms), and what are the connections between them. I want to define a graph.
Having that said, I imagine this application (well, at least that initial part) to be a mix between a Paint and a CMS.

Main part of the ui will be the canvas. An area, on which the drawing happens. Adding new content will happen by clicking on it, and it will be possible to move it around. It should be called `WorldMap`.
Second part of the ui will be the toolbox. As a paint program has different tools to interact with the canvas in various ways, so will be the case here. The tools will be called "modes", as in "modes of operation", they toolbox will be placed below the map, and will be descriptively called `BottomMenu`.
Third part will be on the right of both `WorldMap` and `BottomMenu`, and will be called, of course, `RightMenu`. This is where the details of the currently ongoing work will be placed. In the initial part, this is where the forms will be.

Pardon my poor drawing skills, but this is more or less how I'm imagining the first version to look like:

![a mockup](/the_empire_blog/docs/assets/posts/4/mockup.png)

So without further adue, let's do it!

These should give us some sort of a starting point:
```crystal
# src/the_empire, class TheEmpire

RIGHT_MENU_WIDTH = 400
BOTTOM_MENU_HEIGHT = 120
```

And, we're going to need something like this:

```crystal
# src/the_empire, class TheEmpire#initialize

@world_map = TheEmpire::WorldMap.new(
  position: {0, 0},
  size: {@window_width - RIGHT_MENU_WIDTH, @window_height - BOTTOM_MENU_HEIGHT}
)

@bottom_menu = TheEmpire::BottomMenu.new(
  position: {0, @window_height - BOTTOM_MENU_HEIGHT},
  size: {@window_width - RIGHT_MENU_WIDTH, BOTTOM_MENU_HEIGHT}
)

@right_menu = TheEmpire::RightMenu.new(
  position: {@window_width - RIGHT_MENU_WIDTH, 0},
  size: {RIGHT_MENU_WIDTH, @window_height}
)
```

So it turns our, when working on that app, I need to think a lot about screen real estate. Much more so than when working on a web application, because there, browser solves a lot of problems for me. Main tool to think about where things are placed on a screen is what I came to think of as "bounding rectangle".
Bounding rectangle is a rectangle on a screen. It is defined by 4 numbers: 2 numbers indicating where it's top-left corner is, and 2 numbers indicating it's width and height.

This is exactly what `position` and `size` indicate in the code above. And, since I am a mathematician by education, one more thing that's been surprising for me is this:
point 0, 0 is in top left corner in the screen, the x value increases as we go right, and y value increases as we go down. In pure math, that is handled a bit differentely.

So, the `WorldMap` begins in the top, left corner, and extends all the way to right, ending `RIGHT_MENU_WIDTH` pixels before the right corner, and all the way down, finishing `BOTTOM_MENU_HEIGHT` pixels before the bottom.
`BottomMenu` begins glued to the left screen border, but beginning on top where `WorldMap` ends, and extending to the right in the same way `WorldMap` does.
Finally, `RightMenu` takes the remaining screen size on the right.

Let's see what we need to implement these classess:

```crystal
# src/the_empire/world_map.cr

class TheEmpire
  class WorldMap
    include SF::Drawable

    def initialize(@position : Tuple(Int32, Int32), @size : Tuple(Int32, Int32))
    end

    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      background = SF::RectangleShape.new(@size)
      background.position = @position
      background.fill_color = SF::Color.new(20, 208, 240)

      target.draw(background, states)
    end
  end
end
```

[DF::Drawable](https://oprypin.github.io/crsfml/api/SF/Drawable.html) comes from `CrSFML` directly and is used to make rendering easier.
For now, we just save `@position` and `@size`, and render a red rectangle where the bounding rectangle is. Implementation for `BottomMenu` and `RightMenu` are identical, except I made them different colors.

To finish this part, we need to update the top of `src/the_empire.cr` to:

```crystal

require "crsfml"
require "./the_empire/**"
```

And I have to say, I really love it. The second line here allows us to recursively require an entire directory. Ruby doesn't have that, and it's awesome.

Finally, we need to update the rendering code:

```crystal
# src/the_empire, class TheEmpire

def render
  @window.clear(SF::Color::White)

  @window.draw(@world_map)
  @window.draw(@bottom_menu)
  @window.draw(@right_menu)

  @window.display
end
```

We'll leave the `@shape` out of this for now.
And that gives us this:

![a very bright UI shape](/the_empire_blog/docs/assets/posts/4/basic_shape.png)

We have the structure that we want! I initially made the colors red, green and blue, but that really hurt my eyes after looking at it shortly, so for presentation I picked more "soft" colors : ).

Since I am not a UI designer, nor have any skills for it, for my actual UI, I copy-pasted colors from Factorio:

```crystal
# src/constants.cr

module Constants
  COLOR::MENU::BACKGROUND = SF::Color.new(96, 96, 96)
  COLOR::MENU::BACKGROUND_BORDER = SF::Color.new(227, 227, 227)
end
```

Then require it on top of `src/the_empire.cr`, and use `Constants::COLOR::MENU::BACKGROUND` in `RightMenu` and `BottomMenu`, and white in `WorldMap`. Voila:

![a very grey UI](/the_empire_blog/docs/assets/posts/4/grey_ui.png)

So what do we do with the `@shape` ? It belongs in the `WorldMap`, as it's a stand-in for the content we'll be drawing. I will be moving `@shape` and handling of all events that update it's state to `WorldMap`:

```crystal
# src/the_empire.cr

require "crsfml"

require "./constants"
require "./the_empire/**"

class TheEmpire
  RIGHT_MENU_WIDTH = 400
  BOTTOM_MENU_HEIGHT = 120

  def initialize
    @window_width = 1920
    @window_height = 1080

    @window = SF::RenderWindow.new(SF::VideoMode.new(@window_width, @window_height), "My window")
    # If I don't do that, it actually renders ~20000 FPS, lol
    @window.framerate_limit = 60
    # I have 3 screens, and if I don't set it, it renders in my left-most screen.
    # I would appreciate the option to define main screen, actually.
    @window.position = SF.vector2(6500, 1800)

    @world_map = TheEmpire::WorldMap.new(
      position: {0, 0},
      size: {@window_width - RIGHT_MENU_WIDTH, @window_height - BOTTOM_MENU_HEIGHT}
    )

    @bottom_menu = TheEmpire::BottomMenu.new(
      position: {0, @window_height - BOTTOM_MENU_HEIGHT},
      size: {@window_width - RIGHT_MENU_WIDTH, BOTTOM_MENU_HEIGHT}
    )

    @right_menu = TheEmpire::RightMenu.new(
      position: {@window_width - RIGHT_MENU_WIDTH, 0},
      size: {RIGHT_MENU_WIDTH, @window_height}
    )
  end

  def running?
    @window.open?
  end

  def handle_events
    while event = @window.poll_event
      handle_event(event)
      @world_map.handle_event(event)
    end
  end

  # Handle the close event specifically
  def handle_event(event : SF::Event::Closed)
    @window.close
  end

  # Ignore any other event
  def handle_event(event)
  end

  def update
    @world_map.update
  end

  def render
    @window.clear(SF::Color::White)

    @window.draw(@world_map)
    @window.draw(@bottom_menu)
    @window.draw(@right_menu)

    @window.display
  end
end
```

```crystal
# src/the_empire/world_map.cr

class TheEmpire
  class WorldMap
    include SF::Drawable

    MOVING_SPEED = 10

    property moving_up_speed = 0, moving_left_speed = 0

    def initialize(@position : Tuple(Int32, Int32), @size : Tuple(Int32, Int32))
      @shape = SF::CircleShape.new(300)
      @shape.fill_color = SF::Color::Black

      @moving_around = false
      @mouse_button_initial_x = 0
      @mouse_button_initial_y = 0
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

    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      target.draw(@shape, states)
    end
  end
end
```

And now we have a clearly defined canvas:

<video width="100%" src="/the_empire_blog/docs/assets/posts/4/ui_shape_with_canvas.mp4" controls autoplay></video>

Final thing I want to do at this stage is to make bounding rectangles a first-class citizen. Let's check the code again:

```crystal
class TheEmpire
  class RightMenu
    include SF::Drawable

    def initialize(@position : Tuple(Int32, Int32), @size : Tuple(Int32, Int32))
    end

    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      background = SF::RectangleShape.new(@size)
      background.position = @position
      background.fill_color = Constants::COLOR::MENU::BACKGROUND

      target.draw(background, states)
    end
  end
end
```

Notice we are saving `@position` and `@size` as separate instance variables, and then using them to set the `background` for rendering. But they are representative of a single value: the bounding rectangle. So I would like this code to look like this instead:

```crystal
class TheEmpire
  class RightMenu
    include SF::Drawable

    def initialize(position, size)
      @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])
    end

    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      background = SF::RectangleShape.new(@bounding_rectangle)
      background.fill_color = Constants::COLOR::MENU::BACKGROUND

      target.draw(background, states)
    end
  end
end
```

I want to save the rectangle in instance variable, and use it to create the background. But that raises an error:

```bash
In src/the_empire/bottom_menu.cr:10:39

 10 | background = SF::RectangleShape.new(@bounding_rectangle)
                                      ^--
Error: no overload matches 'SF::RectangleShape.new' with type SF::Rect(Int32)

Overloads are:
 - SF::RectangleShape.new(size : Vector2 | Tuple = Vector2.new(0, 0))
 - SF::RectangleShape.new(copy : RectangleShape)
```

There is no such initializer for `SF::RectangleShape`. Well, easy, I'll just add one:

```crystal
# src/lib/sf/rectangle_shape.cr

class SF::RectangleShape
  # https://github.com/oprypin/crsfml/blob/master/src/graphics/obj.cr#L3962-L3965
  def initialize(bounding_rectangle : SF::IntRect)
    SFMLExt.sfml_rectangleshape_allocate(out @this)
    size = SF.vector2f(bounding_rectangle.size[0], bounding_rectangle.size[1])
    SFMLExt.sfml_rectangleshape_initialize_UU2(to_unsafe, size)

    self.position = bounding_rectangle.position
  end
end
```

Ok, maybe not *that* easy. I don't fully understand what's happening here, but I found the implementation of the already-existing initializer and tweaked it a little bit. I created a `/lib` directory, which I'll use to store tools and extensions to tools, require it in `src/the_empire.cr`, and voila:

```crystal
killa@killa-MS-7A34:~/workspace/the_empire_redone/the_empire$ crystal src/main.cr
Showing last frame. Use --error-trace for full trace.

In src/lib/sf/rectangle_shape.cr:5:43

 5 | size = SF.vector2f(bounding_rectangle.size[0], bounding_rectangle.size[1])
                                           ^---
Error: undefined method 'size' for SF::Rect(Int32)
```

`SF::Rect` doesn't have `#size`. Well, let's add that one as well:

```crystal
# src/lib/sf/rect.cr

struct SF::Rect
  def position
    {@left, @top}
  end

  def position=(new_position)
    @left = new_position[0]
    @top = new_position[1]
  end

  def size
    {@width, @height}
  end

  def size=(new_size)
    @width = new_size[0]
    @height = new_size[1]
  end
end
```

And that does the trick just fine! I updated the `WorldMap` and both menus to use it, and we're done for this one.

In the next post, we will discover what happens when we try to [render more shapes](5-rendering-more-shapes.html)!

