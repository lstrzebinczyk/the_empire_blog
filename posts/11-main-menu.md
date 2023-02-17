Today we will take a small detour. At this time, the application opens directly into the map creator, and the entire codebase is organized around the map creator. I think this is not how it should be.
I would like to have a main menu (like in games), and an option to pick between several pages. We will for sure have an option to run the simulation, but also we will need some sort of page to traverse the data we create. All of these pages need to have some sort of separation. Therefore, today we will introduce the concept of pages, and a new one: the main menu page.

## Map Page

But we will begin by saying this: Everything you can do in the app right now, is a map page. Or `Page::Map`. We will split our `TheEmpire` class into two.

First part will continue to be `TheEmpire` class, but it is now only responsible for initializing a page, and managing the main loop:

```crystal
# src/the_empire.cr

require "crsfml"

require "./lib/**"
require "./constants"
require "./the_empire/**"
require "./page/**"

class TheEmpire
  def initialize
    @window_width = 1920
    @window_height = 1080

    @window = SF::RenderWindow.new(SF::VideoMode.new(@window_width, @window_height), "My window")
    @window.framerate_limit = 60
    @window.position = SF.vector2(6500, 1800)

    @map_page = Page::Map.new(@window_width, @window_height, @window)
  end

  def running?
    @window.open?
  end

  def handle_events
    while event = @window.poll_event
      handle_event(event)
      @map_page.handle_event(event)
    end
  end

  # Handle the close event specifically
  def handle_event(event : SF::Event::Closed)
    @window.close
  end

  def handle_event(event)
  end

  def update
    @map_page.update
  end

  def render
    @window.clear(SF::Color::White)
    @map_page.render
    @window.display
   end
end
```

And the second one will be `Page::Map`. We will move all the business logic we implemented about managing map to this class:

```crystal
# src/page/map.cr

module Page
  class Map
    RIGHT_MENU_WIDTH = 400
    BOTTOM_MENU_HEIGHT = 120

    @active_mode : TheEmpire::Mode::BaseMode

    def initialize(@window_width : Int32, @window_height : Int32, @window : SF::RenderWindow)
      @world_map = TheEmpire::WorldMap.new(
        position: {0, 0},
        size: {@window_width - RIGHT_MENU_WIDTH, @window_height - BOTTOM_MENU_HEIGHT},
        window: @window
      )

      @move_mode = TheEmpire::Mode::MoveMode.new(@world_map)
      @poi_mode = TheEmpire::Mode::POIMode.new(@world_map)

      @modes = [
        @move_mode,
        @poi_mode
      ]

      @active_mode = @move_mode

      @bottom_menu = TheEmpire::BottomMenu.new(
        position: {0, @window_height - BOTTOM_MENU_HEIGHT},
        size: {@window_width - RIGHT_MENU_WIDTH, BOTTOM_MENU_HEIGHT},
        modes: @modes,
        active_mode: @active_mode,
        omit_event: ->handle_page_event(TheEmpire::Event)
      )

      @right_menu = TheEmpire::RightMenu.new(
        position: {@window_width - RIGHT_MENU_WIDTH, 0},
        size: {RIGHT_MENU_WIDTH, @window_height},
        active_mode: @active_mode
      )
    end

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
        @right_menu.active_mode = @active_mode
      end

      nil
    end

    def handle_event(event)
      @active_mode.handle_event(event)
      @bottom_menu.handle_event(event)
      @right_menu.handle_event(event)
    end

    def render
      @window.draw(@world_map)
      @window.draw(@active_mode)
      @window.draw(@bottom_menu)
      @window.draw(@right_menu)
    end

    def update
      @world_map.update
    end
  end
end
```

Easy peasy, nothing here is new. We just move the code around. We are also off-screen going to move the entire contents of `/src/the_empire` directory to `/src/page/map` and rename them accordingly.
Whole `/src/the_empire` is about working with the map, so this is keeping in line with encapsulating this logic in the new page.

I have to say, this is a very small change, but it makes me very happy and it makes the code seem much better organized: )

## Menu class

If we want to make a new page, it must respond to the same methods, if we want it to be a drop-in replacements. Let's try this:

```crystal
# src/page/menu.cr

module Page
  class Menu
    @ui : UI::Item

    def initialize(@window_width : Int32, @window_height : Int32, @window : SF::RenderWindow)
      @bounding_rectangle = SF::IntRect.new(0, 0, @window_width, @window_height)

      @ui = build_ui()
    end

    def handle_event(event : SF::Event)
      @ui.handle_event(event)
    end

    def update
    end

    def render
      @window.draw(@ui)
    end

    private def build_ui
      UI::Box.new(@bounding_rectangle) do |c|
        c.vertical(gap: 50) do |c|
          c.button(
            size: {300, 80},
            text: "Map",
            on_click: ->{
              p "map"
            },
          )
          c.button(
            size: {300, 80},
            text: "Exit",
            on_click: ->{
              p "exit"
            },
          )
        end
      end.background(fill_color: Constants::COLOR::MENU::BACKGROUND)
    end
  end
end
```

If we do this little trick:

```crystal
# src/the_empire.cr, class TheEmpire#initialize

# just temporarily comment out the map page and replace it with a menu page
# @map_page = Page::Map.new(@window_width, @window_height, @window)
@map_page = Page::Menu.new(@window_width, @window_height, @window)
```

We will see this, when we open the app:

![menu page](/the_empire_blog/docs/assets/posts/11/menu_1.png)

Awesome! But I think I would like these buttons to be aligned to the top a little bit. Let's put 1 spacer above the buttons, and 2 below:

```crystal
# src/page/menu.cr, class TheEmpire#build_ui

# The spacers are new
UI::Box.new(@bounding_rectangle) do |c|
  c.vertical(gap: 50) do |c|
    c.spacer
    c.button(
      size: {300, 80},
      text: "Map",
      on_click: ->{
        p "map"
      },
    )
    c.button(
      size: {300, 80},
      text: "Exit",
      on_click: ->{
        p "exit"
      },
    )
    c.spacer
    c.spacer
  end
end.background(fill_color: Constants::COLOR::MENU::BACKGROUND)
```

Now let's check it:

![menu page with better positioning](/the_empire_blog/docs/assets/posts/11/menu_2.png)

Much better! We definitely want to start our app from that page. We would like the "Map" button to change the page to map, and the "Exit" button to close the program.
We will deal with it similarily to how we deal with the modes.

First of all, we will actually initialize both pages and have an `@active_page`:

```crystal
# src/page/menu.cr, class TheEmpire#initialize

@map_page = Page::Map.new(@window_width, @window_height, @window)
@menu_page = Page::Menu.new(@window_width, @window_height, @window)
@active_page = @menu_page
```

And in addition to that, we will update the rest of the class to work with `@active_page`. Now, we want to be able to change pages. The problem is that this can only happen as a result of some action that happens inside of a page, but the variables that control it are in the layer above the pages. We will introduce a concept similiar to what we did in the map page. There, we have a `handle_page_event` method, which can work with `Page::Map::Event`, and in here we will have a `handle_app_event` which will be able to handle `TheEmpire::Event`. Let's see:

```crystal
# src/event.cr

class TheEmpire
  abstract struct Event
    struct ChangePageEvent < Event
      getter page

      def initialize(@page : Symbol)
      end
    end
  end
end
```

Easy enough, we define an event class which indicates, that a page should change. Next step:

```crystal
# src/the_empire.cr, class TheEmpire

def handle_app_event(event : TheEmpire::Event)
  case event
  when TheEmpire::Event::ChangePageEvent
    case event.page
    when :map
      @active_page = @map_page
    end
  end

  nil
end
```

Should our main loop class be asked to handle an event, it will be able to. If that event is a `ChangePageEvent` with a `:map` for a `#page`, it will change the `@active_page` to `@map_page`. Easy.
We pass that function as a proc to page objects:

```crystal
# src/the_empire.cr, class TheEmpire#initialize

# the final `omit_event` argument is new
@map_page = Page::Map.new(@window_width, @window_height, @window, omit_event: ->handle_app_event(TheEmpire::Event))
@menu_page = Page::Menu.new(@window_width, @window_height, @window, omit_event: ->handle_app_event(TheEmpire::Event))
```

And we'll accept that event and use it in the `Page::Menu`:

```crystal
# src/page/menu.cr, class Page::Menu

# The final argument is new
def initialize(@window_width : Int32, @window_height : Int32, @window : SF::RenderWindow, @omit_event : Proc(TheEmpire::Event, Nil))
  @bounding_rectangle = SF::IntRect.new(0, 0, @window_width, @window_height)

  @ui = build_ui()
end
```

And finally, we'll use it:

```crystal
# src/page/menu.cr, class Page::Menu

private def build_ui
  UI::Box.new(@bounding_rectangle) do |c|
    c.vertical(gap: 50) do |c|
      c.spacer
      c.button(
        size: {300, 80},
        text: "Map",
        on_click: ->{
          # Contents of this block are new. We create an event object and call the `@omit_event` proc with it.
          event = TheEmpire::Event::ChangePageEvent.new(:map)
          @omit_event.call(event)
        },
      )
      c.button(
        size: {300, 80},
        text: "Exit",
        on_click: ->{
          p "exit"
        },
      )
      c.spacer
      c.spacer
    end
  end.background(fill_color: Constants::COLOR::MENU::BACKGROUND)
end
```

And there we have it:

<video width="100%" src="/the_empire_blog/docs/assets/posts/11/working_button.mp4" controls autoplay></video>

It does have some issues, but it definitely works. When we press the `exit` button, we want the program to close, but that is trivial and I'll do it off-screen.
For now, this is what we wanted. In the next one, we'll [make the map look more like a map](12-make-it-look-more-like-a-map.html)!
