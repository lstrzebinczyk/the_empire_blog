Let me start by giving a small disclaimer. I'm fully aware that the UI system I'm implementing here is far from being a nice one.
The UI system I'll be presenting over the several next posts should be good enough to get this project to it's first milestone. Once that is done, we'll probably implement a better one.
But for now...

## A font

[The docs](https://oprypin.github.io/crsfml/api/SF/Text.html) tell us, that in order to render any text, we need to begin by having a font. I'll fetch [titilium web](https://fonts.google.com/specimen/Titillium+Web), put it into `./assets/fonts/titilium`, and update the constants like so:

```crystal
# src/constants.cr

module Constants
  FONT = SF::Font.from_file("./assets/fonts/titilium/TitilliumWeb-Regular.ttf")

  COLOR::MENU::BACKGROUND = SF::Color.new(96, 96, 96)
  COLOR::MENU::BACKGROUND_BORDER = SF::Color.new(227, 227, 227)
end
```

And we have a font now. Let's move on to...

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
      # Define a grey rectangle to be rendered at @bounding_rectangle
      background = SF::RectangleShape.new(@bounding_rectangle)
      background.fill_color = SF::Color.new(142, 142, 142)

      # Define a text object. We needed to have a font object first, because it's needed here
      text = SF::Text.new(@text, Constants::FONT, 60)
      text.color = SF::Color::Black
      text.position = @bounding_rectangle.position

      # Render both
      target.draw(background, states)
      target.draw(text, states)
    end
  end
end
```

We're going to make a good use of it in the bottom menu:

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

That's a button, allright. Mission accomplished. It needs several improvements, though:
- It should be vertically centered, and should have a padding on the left
- the text should be centered
- it needs to respond to being clicked

## Centering the button inside the container

The first part we can cheat for now. The bottom menu has 120px of height, and the button has 80px of height.
If we render it 20px to the right and to the bottom, from it's current position, that should be just fine. I'll just update the button definition to this:

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

Nice and centered. We have successfully avoided the necessity to think (about that specific problem, for now).

## Centering text inside button

That part, we actually need to think through a little bit. We'll be adding a lot of different buttons, and they will be allowed to have different sizes, font sizes, and have different texts. For all of them, the text should be both vertically and horizontally centered.
Since text is basically a rectangle, center one within the bounding rectangle should totally, really be fairly simple.

If we take the `background` `left` position, and add half of the `background` width, we should get the middle of the buttons `x` position. If we then substract half of the `text` width, that should give us the expected `x` position. Analogous solution for height should workas well.

That's easy math, 20 minutes adventure, in and out:

```crystal
# src/lib/ui/button.cr, class UI::Button

def draw(target : SF::RenderTarget, states : SF::RenderStates)
  background = SF::RectangleShape.new(@bounding_rectangle)
  background.fill_color = SF::Color.new(142, 142, 142)

  text = SF::Text.new(@text, Constants::FONT, 60)
  text.color = SF::Color::Black
  text_position_x = @bounding_rectangle.left + @bounding_rectangle.width / 2 - text.local_bounds.width / 2
  text_position_y = @bounding_rectangle.top + @bounding_rectangle.height / 2 - text.local_bounds.height / 2
  text.position = { text_position_x, text_position_y }

  target.draw(background, states)
  target.draw(text, states)
end
```

And that should do just fine! Let's see:

![not centered button at all](/the_empire_blog/docs/assets/posts/6/button_not_centered_at_all.png)

This is not centered at all.

Like I mentioned in the beginning, I am re-creating my steps from a while back with these posts, and that specific issue gives me painful flashbacks. It's a very interesting experience, rediscovering bugs that I already fixed weeks ago. Ones I was hoping never to see again. But well, here we are: We are asking the text to be rendered within a very specific bounding rectangle, and it is instead being rendered quite a good bit off.

I have spent good, long, painful hours trying to understand that bit, but it turns out to be the expected behavior. Managing text on a screen is **really** difficult, and people way smarter than me decided that's a good default behavior.
I have later learned that those people were right, but we'll get there in due time.

Long story short, to do what we want, we need to not only use texts `width` and `height`, but also `top` and `left`. Texts bounds are nice enough to tell us exactly how much they are off:

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

Figuring this one out was especially tricky since it's not explained at all in the [CrSGML Text doc](https://oprypin.github.io/crsfml/api/SF/Text.html),
and I only learned what's happening, and why it is happening after reading [a random forum thread](https://en.sfml-dev.org/forums/index.php?topic=24026.0).

Anyway, that works like expected:

![proper button](/the_empire_blog/docs/assets/posts/6/button_proper.png)

And that takes us to the really tricky part. Making that button react to clicks will require us to think a little bit about [events management](7-events-management.html)!

Allons-y!
