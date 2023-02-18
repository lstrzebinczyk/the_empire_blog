Today we will take a small detour. Currently, when the app starts, it opens map editor. The entire codebase is organized around it.

That is not exactly how it should be.

I envision a main menu, like games have, and several options to choose from. Map editor is one of them, but there will be others. For sure at least a separate page to run the simulation, and a different one to allow easy reviewing of data we create.

We need to have an architecture that supports the idea of separate "pages". We'll try to do that today, and add the first new page: the main menu.

## Map Page

We will start by recognizing, that everything available right now is a map editor. Or a "map page". Or `Page::Map`. Our `TheEmpire` class actually has 2 responsibilities: managing the main loop and managing the contents and logic of map editing. We will split it into 2 classess with more focused responsibilities.

First, we will introduce the new class: `Page::Map`. It's role is to store data related to map management and manage user input. I suppose it is similiar to `C` in the `MVC`.

Everything related to map management from `TheEmpire` is now moved here:

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

We just moved the code around.

Also I am going to move the entire contents of `/src/the_empire` directory to `/src/page/map` and rename them accordingly. So, for example, `TheEmpire::RightMenu` becomes `Page::Map::RightMenu`. That makes it clear that this entire logic is related to map page.

Secondly, what remains in `TheEmpire` class is now responsible for initializing of the entire application, initializing a page, and delegating all actions to the page:

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

    # This is new. We expect it to cover map page behavior entirely, so we ask it to...
    @map_page = Page::Map.new(@window_width, @window_height, @window)
  end

  def running?
    @window.open?
  end

  def handle_events
    while event = @window.poll_event
      handle_event(event)
      # 1: handle events
      @map_page.handle_event(event)
    end
  end

  def handle_event(event : SF::Event::Closed)
    @window.close
  end

  def handle_event(event)
  end

  def update
    # 2: update itself
    @map_page.update
  end

  def render
    @window.clear(SF::Color::White)
    # 3: render to screen
    @map_page.render
    @window.display
   end
end
```

I have to say, this is a rather a small change in architecture, but it makes the code seem organized much better.

Naturally, the idea is that we can have more pages. We will introduce a new one with...

## Main Menu

Let's try this:

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

It's a very simple page: render 2 buttons and attach actions to them.

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

UI::Box.new(@bounding_rectangle) do |c|
  c.vertical(gap: 50) do |c|
    c.spacer # <-
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
    c.spacer # <-
    c.spacer # <-
  end
end.background(fill_color: Constants::COLOR::MENU::BACKGROUND)
```

Now let's check it:

![menu page with better positioning](/the_empire_blog/docs/assets/posts/11/menu_2.png)

I don't know why, but that feels better.

We want the organization here to be like this:
1. When app starts, we see the menu page
2. Clicking on `Map` button opens the map page
3. Clicking on `Exit` button closes the program.

We will initialize both pages and set the menu one as active:

```crystal
# src/page/menu.cr, class TheEmpire#initialize

@map_page = Page::Map.new(@window_width, @window_height, @window)
@menu_page = Page::Menu.new(@window_width, @window_height, @window)
@active_page = @menu_page
```

The rest of this class will now work with `@active_page`. We are storing information about which page is active inside the `TheEmpire` class, but we want it to change based on a button press inside a `Page::Menu`, which is a "subcomponent" of `TheEmpire`. We already solved that problem with modes on the page map, we'll replicate that solution here.

Let's introduce an app-level event:

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

We implement a method to handle the event. Should it be calles with `TheEmpire::Event::ChangePageEvent`, it will try and change the active page.

We pass that function as a proc to page objects:

```crystal
# src/the_empire.cr, class TheEmpire#initialize

# the final `omit_event` argument is new
@map_page = Page::Map.new(@window_width, @window_height, @window, omit_event: ->handle_app_event(TheEmpire::Event))
@menu_page = Page::Menu.new(@window_width, @window_height, @window, omit_event: ->handle_app_event(TheEmpire::Event))
```

We'll accept that proc and save it in the `Page::Menu`:

```crystal
# src/page/menu.cr, class Page::Menu

# The final argument is new
def initialize(@window_width : Int32, @window_height : Int32, @window : SF::RenderWindow, @omit_event : Proc(TheEmpire::Event, Nil))
  @bounding_rectangle = SF::IntRect.new(0, 0, @window_width, @window_height)

  @ui = build_ui()
end
```

And finally, we'll use it when the "Map" button is pressed.

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

Let's traverse this logic from the back.
1. When initializing `Page` classess, we give them a proc (a function), which will accept some hardcoded events.
1. We implement a "Please change the page" event, the `TheEmpire::Event::ChangePageEvent`
1. When clicking on the "Map" button in main menu, we create an instance of `TheEmpire::Event::ChangePageEvent` event and call the event-processing proc with it
1. Parent recognizes the event and updates the `@active_page` to `@map_page.

And there we have it:

<video width="100%" src="/the_empire_blog/docs/assets/posts/11/working_button.mp4" controls autoplay></video>

Video is quite short, but it does showcase that it is working. It does have some issues, but we'll ignore them for now.

Second button has quite a similiar story: we create an `TheEmpire::Event::ExitAppEvent` event, and when `#handle_app_event` receives it, the same as when we press the escape key.
Since that is quite trivial, I'll do that off-screen.

We have been playing with technical improvements for a while now, it's time to push some features. In the next one, we will be [introducing the database](12-introducing-the-database.html)!
