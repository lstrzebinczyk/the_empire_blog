---
title: "Humble beginning"
date: 2019-01-20
---

Well, all great adventures begin with `~rails new~` `crystal init`! So let's see:

```bash
  killa@killa-MS-7A34:~/workspace$ crystal init app the_empire
    create  /home/killa/workspace/the_empire/.gitignore
    create  /home/killa/workspace/the_empire/.editorconfig
    create  /home/killa/workspace/the_empire/LICENSE
    create  /home/killa/workspace/the_empire/README.md
    create  /home/killa/workspace/the_empire/shard.yml
    create  /home/killa/workspace/the_empire/src/the_empire.cr
    create  /home/killa/workspace/the_empire/spec/spec_helper.cr
    create  /home/killa/workspace/the_empire/spec/the_empire_spec.cr
  Initialized empty Git repository in /home/killa/workspace/the_empire/.git/
```

hooray!

So, what we have is

```crystal
# src/the_empire.cr

# TODO: Write documentation for `TheEmpire`
module TheEmpire
  VERSION = "0.1.0"

  # TODO: Put your code here
end
```

Since it needs to be a visual app, I chose to use [CrSFML](https://oprypin.github.io/crsfml/). It seems to give me what I'm going to need.
Adding to `shards.yml`:

```
dependencies:
  crsfml:
    github: oprypin/crsfml
    version: ~> 2.5.2
```

Running `shards install`:

```bash
killa@killa-MS-7A34:~/workspace/the_empire$ shards install
Resolving dependencies
Fetching https://github.com/oprypin/crsfml.git
Installing crsfml (2.5.2)
Postinstall of crsfml: make
Writing shard.lock
```

So far, so good.

[Tutorial](https://oprypin.github.io/crsfml/tutorials/window/window.html) gives us this:

```crystal
require "crsfml"

window = SF::RenderWindow.new(SF::VideoMode.new(800, 600), "My window")

# run the program as long as the window is open
while window.open?
  # check all the window's events that were triggered since the last iteration of the loop
  while event = window.poll_event
    # "close requested" event: we close the window
    if event.is_a? SF::Event::Closed
      window.close
    end
  end
end
```

Let's decompose this:
- I need to create an instance of `SF::RenderWindow`
- I need to create an infinite loop, running while that window is `#open?`
- in my infinite loop, I need to poll events from that window
- if one of that events is of type `SF::Event::Closed`, I need to close that window

That's very interesting! And very different to webdev, which I'm used to. I'll change `TheEmpire` to be a class like so:

```crystal
# src/the_empire.cr

require "crsfml"

class TheEmpire
  def initialize
    @window_width = 1920
    @window_height = 1080

    @window = SF::RenderWindow.new(SF::VideoMode.new(@window_width, @window_height), "My window")
    # If I don't do that, it actually renders ~20000 FPS, lol
    @window.framerate_limit = 60
    # I have 3 screens, and if I don't set it, it renders in my left-most screen.
    # I would appreciate the option to define main screen, actually.
    @window.position = SF.vector2(6500, 1800)
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

  # Ignore any other event
  def handle_event(event)
  end
end
```

And a quick trip to `crystal src/main.cr` gives us this:

![Empty black screen](the_empire_blog/docs/assets/posts/2/empty_black_screen.png)

Nice. Why is it black? [A tutorial page on drawing](https://oprypin.github.io/crsfml/tutorials/graphics/draw.html) contains an extension of the code we were supposed to write last time:

```crystal
  # clear the window with black color
  window.clear(SF::Color::Black)

  # draw everything here...
  window.draw(...)

  # end the current frame
  window.display
```

What I'll do, is I'll extend `TheEmpire` class with:

```crystal
# src/the_empire.cr, class TheEmpire

def render
  @window.clear(SF::Color::White)

  shape = SF::CircleShape.new(300)
  shape.fill_color = SF::Color::Black

  @window.draw(shape)

  @window.display
end
```

and update the main loop to:

```crystal
# src/main.cr

require "./the_empire"

the_empire = TheEmpire.new

while the_empire.running?
  the_empire.handle_events
  the_empire.render
end
```

And that will give us

![A white screen, and a black ball](/the_empire_blog/docs/assets/posts/2/nonempty_white_screen.png)

We have successfully rendered something! That will be enough for today.
