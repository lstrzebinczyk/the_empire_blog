Let me start by giving a small disclaimer. I'm fully aware that the UI system I'm implementing here is far from being a nice one.
The UI system I'll be presenting over the several next posts should be good enough to get this project to it's first milestone. Once that is done, we'll probably implement a better one.
But for now...

## A font

(The docs)[https://oprypin.github.io/crsfml/api/SF/Text.html] tell us, that in order to render any text, we need to begin by having a font. I'll fetch (titilium web)[https://fonts.google.com/specimen/Titillium+Web], put it into `./assets/fonts/titilium`, and update the constants like so:

```crystal
# src/constants.cr

module Constants
  FONT = SF::Font.from_file("./assets/fonts/titilium/TitilliumWeb-Regular.ttf")

  COLOR::MENU::BACKGROUND = SF::Color.new(96, 96, 96)
  COLOR::MENU::BACKGROUND_BORDER = SF::Color.new(227, 227, 227)
end
```

And move on.

## Implementing a button

So what is a button? That's easy, button is something that:
- has a position on screen
- has specific size
- shows a text
- does something when clicked

A simple, initial, implementation could look like this:

```crystal
# src/lib/ui/button.cr

module UI
  class Button
    @bounding_rectangle : SF::IntRect
    @text : String
    @on_click : Proc(Nil)

    def initialize(position, size, text, on_click)
      # we combine position and size to produce @bounding_rectangle
      @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])

      # we save the buttons text and action
      @text = text
      @on_click = on_click
    end

    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      # We define a grey rectangle to be rendered at @bounding_rectangle
      background = SF::RectangleShape.new(@bounding_rectangle)
      background.fill_color = SF::Color.new(142, 142, 142)

      # we define a text object
      text = SF::Text.new(@text, Constants::FONT, 60)
      text.color = SF::Color::Black
      text.position = @bounding_rectangle.position

      # and render them both
      target.draw(background, states)
      target.draw(text, states)
    end
  end
end
```

And let's create one in the bottom menu:

```crystal
# src/the_empire/bottom_menu.cr

class TheEmpire
  class BottomMenu
    include SF::Drawable

    def initialize(position, size)
      @bounding_rectangle = SF::IntRect.new(position[0], position[1], size[0], size[1])
    end

    def draw(target : SF::RenderTarget, states : SF::RenderStates)
      background = SF::RectangleShape.new(@bounding_rectangle)
      background.fill_color = Constants::COLOR::MENU::BACKGROUND

      button = UI::Button.new(
        position: @bounding_rectangle.position,
        size: {200, 80},
        text: "btn",
        on_click: -> { p "Click!" }
      )

      target.draw(background, states)
      target.draw(button, states)
    end
  end
end
```

And there we have it:

![A button](/the_empire_blog/docs/assets/posts/6/button.png)

That's a button, allright. It needs several improvements:
- It should be vertically centered, and should have a padding on the left
- the text should be centered
- it needs to respond to being clicked

## Centering the button inside the container

The first part we can cheat for now. The bottom menu has 120px of height, and the button has 80px of height.
If we render it 20px to the right and to the bottom, it should have a nice visual placement. I'll just update the button definition to this:

```crystal
# src/the_empire/bottom_menu.cr, class TheEmpire::BottomMenu#draw

button = UI::Button.new(
  position: @bounding_rectangle.position.map {|v| v + 20}, # update both position values to be 20 larger
  size: {200, 80},
  text: "btn",
  on_click: -> { p "Click!" }
)
```

Which gives us this:

![A centered button](/the_empire_blog/docs/assets/posts/6/button_centered.png)

Nice and centered. We have successfully avoided the necessity to think.

## Centering text inside button

That part, we actually need to think through a little bit. We'll be adding a lot of different buttons, and they will be allowed to have different sizes, font sizes, and have different texts. For all of them, the text should be both vertically and horizontally centered.
Text is basically a rectangle, so to center one within the other should really be fairly simple.
If we take the position, of the `background`, and add to it the half of the width of the `background`, we should get the middle of the button. If we then substract half of the length of the `text`, that should give us the expected position for the text. Analogous solution for height should workas well. Ok, easy math, in and out:

```crystal
# src/lib/ui/button.cr, class UI::Button

def draw(target : SF::RenderTarget, states : SF::RenderStates)
  background = SF::RectangleShape.new(@bounding_rectangle)
  background.fill_color = SF::Color.new(142, 142, 142)

  text = SF::Text.new(@text, Constants::FONT, 60)
  text.color = SF::Color::Black
  text_position_x = (@bounding_rectangle.left + @bounding_rectangle.width / 2 - text.local_bounds.width / 2).to_i
  text_position_y = (@bounding_rectangle.top + @bounding_rectangle.height / 2 - text.local_bounds.height / 2).to_i
  text.position = { text_position_x, text_position_y }

  target.draw(background, states)
  target.draw(text, states)
end
```

![not centered button at all](/the_empire_blog/docs/assets/posts/6/button_not_centered.png)

This is not centered at all, and I already regret reliving this experience. It's a very interesting experience, rediscovering bugs that I already fixed weeks ago.
The text in here is not rendered inside the expected position at all, and that's expected behavior. Turns out managing text on a screen is **really** difficult, and people way smarter than me picked the default behavior in here.
To do what we want, we need to make use of all the texts local bounds:

```crystal
# src/lib/ui/button.cr, class UI::Button

def draw(target : SF::RenderTarget, states : SF::RenderStates)
  background = SF::RectangleShape.new(@bounding_rectangle)
  background.fill_color = SF::Color.new(142, 142, 142)

  text = SF::Text.new(@text, Constants::FONT, 60)
  text.color = SF::Color::Black
  text_position_x = (@bounding_rectangle.left + @bounding_rectangle.width / 2 - text.local_bounds.width / 2 - text.local_bounds.left).to_i
  text_position_y = (@bounding_rectangle.top + @bounding_rectangle.height / 2 - text.local_bounds.height / 2 - text.local_bounds.top).to_i
  text.position = { text_position_x, text_position_y }

  target.draw(background, states)
  target.draw(text, states)
end
```

It took me long, long hours to understand that behavior, which is especially tricky since it's not explained at all in the [CrSGML Text doc](https://oprypin.github.io/crsfml/api/SF/Text.html),
and I only learned what's happening here and why after reading [a random forum thread](https://en.sfml-dev.org/forums/index.php?topic=24026.0).
Anyway, that works:

![proper button](/the_empire_blog/docs/assets/posts/6/button_proper.png)

And that takes us to the really tricky part.

## Events management
