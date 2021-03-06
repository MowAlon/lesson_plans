---
title: Cart Implementation
length: 120
tags: cart, order
---

## Goals

* represent a cart using a PORO in Rails
* uses a `flash` to send messages to the view
* load an object to be used throughout the app using a `before_action` in the ApplicationController

## Structure

* Code-along

## Video

* [Cart and Order Lifecycle](https://vimeo.com/126190416)

## Repository

* [Catch 'Em All](https://github.com/rwarbelow/catch-em-all)

## Intro

We'll build out an app where the user should capture pokemon that are added to his/her backpack. The captured pokemon are not saved in the database until the user has chosen to end the game. 

This will mimick the cart-order lifecycle. Pokemon are the items, the backpack is the cart, and an order is the saved game. 

## Code-Along

* `git clone https://github.com/rwarbelow/catch-em-all.git`
* `rake db:create db:migrate db:seed`
* `rails s`
* navigate to `http://localhost:3000`
* Want hard mode? Uncomment the JS file to make your Pokemon run around. 

### Writing a Test

* `rails g integration_test user_capture_pokemon`
* Navigate to `test/integration/user_capture_pokemon_test.rb`
* Inside of that file:

```ruby
require 'test_helper'

class UserCapturePokemonTest < ActionDispatch::IntegrationTest
  test "displays capture message and backpack content" do
    Pokemon.create(name: "Pikachu",
                   image_url: "http://core.dawnolmo.com/50_pokemon__9_pikachu_by_megbeth-d5fga3f_original.png")

    visit root_path

    assert page.has_content?("Backpack: 0")

    click_button "Capture"

    assert page.has_content?("You now have 1 Pikachu.")
    assert page.has_content?("Backpack: 1")
  end
end

```

It fails on line 10, which is looking for "Backpack: 0". Let's hard-code that in for now in the layout application.html.erb.

```erb
Backpack: 0
```

Run the test again, and it complains about not having a capture button. Let's make a button and talk about what the path should be. Inside of views/pokemons/index.html.erb:

```erb
  <%= button_to "Capture", backpack_pokemons_path, class: "btn btn-danger" %>
```

Our error now is:

```
  1) Error:
UserCapturePokemonTest#test_displays_capture_message_and_backpack_content:
ActionView::Template::Error: undefined local variable or method `backpack_pokemons_path' for #<#<Class:0x000001015c21d0>:0x000001015caad8>
```

Inside of routes.rb:

```ruby
  resources :backpack_pokemons, only: [:create]
```

Run `rake routes` to see what route this line gives us. Then run the test again. The error is:

```
  1) Error:
UserCapturePokemonTest#test_displays_capture_message_and_backpack_content:
ActionController::RoutingError: uninitialized constant BackpackPokemonsController
```

Make a controller: `touch app/controllers/backpack_pokemons_controller.rb`. If you run the test again, it will complain about missing action `create`. So, inside of the controller file:

```ruby
class BackpackPokemonsController < ApplicationController
  def create
    redirect_to root_path
  end
end
```

Does it work? Start up your server and click the "Capture" button. Does it redirect you back to the same page? Good. 

When you run the test, it fails on line 14, which is looking for the content "You now have 1 Pikachu." This should be a flash message, but right now we don't know which pokemon was caught when we click the capture button. How do we pass a Pokemon ID to the create action? Let's modify our button:

```erb
  <%= button_to "Capture", backpack_pokemons_path(pokemon_id: pokemon.id), class: "btn btn-danger" %>
```

And modify our `create` action in the controller:

```ruby
class BackpackPokemonsController < ApplicationController
  include ActionView::Helpers::TextHelper
  def create
    pokemon = Pokemon.find(params[:pokemon_id])
    flash[:notice] = "You now have #{pluralize(1, pokemon.name)}."
    redirect_to root_path
  end
end
```

And display the flash notice using a content tag in the application.html.erb:

```erb
<% flash.each do |type, message| %>
    <%= content_tag :div, message %>
  <% end %>
```

Great! That line is passing. But now we're failing where it should say that we have 1 pokemon in our backpack. Let's talk about how we can implement this:

* need to store state (which types pokemon have been caught and how many of each type)
* we don't want to save this in our database
* can we store data in a session?


Let's go back into the BackpackPokemonsController:

```ruby
class BackpackPokemonsController < ApplicationController
  include ActionView::Helpers::TextHelper

  def create
    pokemon = Pokemon.find(params[:pokemon_id])
    backpack = session[:backpack] || {}
    backpack[pokemon.id] ||= 0
    backpack[pokemon.id] += 1
    session[:backpack] = backpack
    flash[:notice] = "You now have #{pluralize(1, pokemon.name)}."
    redirect_to root_path
  end
end
```

And in the layout:

```erb
Backpack: <%= session[:backpack].values.sum %>
```

But when you run your tests, you'll see that we're calling `values` on nil. This is because `session[:backpack]` is nil the first time around before we hit the create method. 

And it's probably better to extract this functionality to a PORO instead anyway. Let's refactor our controller:

```ruby
class BackpackPokemonsController < ApplicationController
  include ActionView::Helpers::TextHelper

  def create
    pokemon = Pokemon.find(params[:pokemon_id])
    @backpack.add_pokemon(pokemon.id)
    session[:backpack] = @backpack.contents
    flash[:notice] = "You now have #{pluralize(1, pokemon.name)}."
    redirect_to root_path
  end
end
```

And remove that line calling `session[:backpack].values.sum` in your layout.

Now run `rake test`. You should see this error:

```
UserCapturePokemonTest#test_displays_capture_message_and_backpack_content:
NoMethodError: undefined method `add_pokemon' for nil:NilClass
```

That's because we have this variable `@backpack` but it's not defined anywhere. In an online store, you'd want the cart available anywhere in your application. So let's define this variable in the application controller:

```ruby
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception

  before_action :set_backpack

  def set_backpack
    @backpack = Backpack.new(session[:backpack])
  end
end
```

Run the tests. Now we see:

```
  1) Error:
UserCapturePokemonTest#test_displays_capture_message_and_backpack_content:
NameError: uninitialized constant ApplicationController::Backpack
```

We don't have a Backpack object. This will be a PORO, and it won't inherit from ActiveRecord since we're not doing anything database-y with it. Let's create it: `touch app/models/backpack.rb` and define the class:

```ruby
class Backpack
end
```

Run the tests. We are passing in an argument, but our Backpack doesn't accept one currently. At this point, let's drop down to the model level and write a test for what the Backpack should be able to do. `touch test/models/backpack_test.rb` and add the following test:

```ruby
class BackpackTest < ActiveSupport::TestCase
  test "has initial contents" do
    backpack = Backpack.new({ "1" => 1 })

    assert_equal({ "1" => 1 }, backpack.contents)
  end
end
```

Run just that test: `rake test test/models/backpack_test.rb`. Still wrong number of arguments. Let's add an intialize method and an attr_reader:

```ruby
class Backpack
  attr_reader :contents
  def initialize(initial_contents)
    @contents = initial_contents || {}
  end
end
```

Great. The test passes. Now let's run our whole test suite. We get another error: 

```
  1) Error:
UserCapturePokemonTest#test_displays_capture_message_and_backpack_content:
NoMethodError: undefined method `add_pokemon' for #<Backpack:0x00000101039c90 @contents=nil>
```

So let's drop down to the model level again, and write a test for how this method should operate:

```ruby
class BackpackTest < ActiveSupport::TestCase
  test "has initial contents" do
    backpack = Backpack.new({ "1" => 1 })

    assert_equal({ "1" => 1 }, backpack.contents)
  end

  test "can add a pokemon" do
    backpack = Backpack.new({ "1" => 1})

    backpack.add_pokemon(1)
    backpack.add_pokemon(2)

    assert_equal({ "1" => 2, "2" => 1}, backpack.contents)
  end
end
```

We get an undefined method. Let's define the method in Backpack:

```ruby
class Backpack
  attr_reader :contents
  def initialize(initial_contents)
    @contents = initial_contents || {}
  end

  def add_pokemon(pokemon_id)
    contents[pokemon_id.to_s] ||= 0
    contents[pokemon_id.to_s] += 1
  end
end
```

That test passes. Now run the whole test suite. We're back to failing where it asks for the total number of pokemon in the backpack (line 15). We need a way to call something like `@backpack.total`. Let's write a model test:

```ruby
class BackpackTest < ActiveSupport::TestCase
  test "has initial contents" do
    backpack = Backpack.new({ "1" => 1 })

    assert_equal({ "1" => 1 }, backpack.contents)
  end

  test "can add a pokemon" do
    backpack = Backpack.new({ "1" => 1})

    backpack.add_pokemon(1)
    backpack.add_pokemon(2)

    assert_equal({ "1" => 2, "2" => 1}, backpack.contents)
  end

  test "returns total number of all captured pokemon" do
    backpack = Backpack.new({ "1" => 3, "2" => 1, "3" => 3})

    assert_equal 7, backpack.total
  end
end
```

We get an error that we don't have a total method. Implement it:

```ruby
class Backpack
  attr_reader :contents
  def initialize(initial_contents)
    @contents = initial_contents || {}
  end

  def add_pokemon(pokemon_id)
    contents[pokemon_id.to_s] ||= 0
    contents[pokemon_id.to_s] += 1
  end

  def total
    contents.values.sum
  end
end
```

That model test passes. Now we should be able to replace the bit in our application.html.erb layout:

```erb
Backpack: <%= @backpack.total %>
```

And all tests are passing. Our test isn't super robust, though. If you look closely, we're hard-coding in `1` in our controller in the pluralize method:

```ruby
class BackpackPokemonsController < ApplicationController
  include ActionView::Helpers::TextHelper
  def create
    pokemon = Pokemon.find(params[:pokemon_id])
    @backpack.add_pokemon(pokemon.id)
    session[:backpack] = @backpack.contents
    flash[:notice] = "You now have #{pluralize(1, pokemon.name)}."
    redirect_to root_path
  end
end
```

So let's make our test more robust:

```ruby
require 'test_helper'

class UserCapturePokemonTest < ActionDispatch::IntegrationTest
  test "displays capture message and backpack content" do
    Pokemon.create(name: "Pikachu",
                   image_url: "http://core.dawnolmo.com/50_pokemon__9_pikachu_by_megbeth-d5fga3f_original.png")

    visit root_path

    assert page.has_content?("Backpack: 0")

    click_button "Capture"

    assert page.has_content?("You now have 1 Pikachu.")
    assert page.has_content?("Backpack: 1")

    click_button "Capture"

    assert page.has_content?("You now have 2 Pikachus.")
    assert page.has_content?("Backpack: 2")
  end
end
```

Now it fails on line 19. If you do a `save_and_open_page`, you'll see that it says "Backpack: 2" which is correct, but the flash still says "You now have 1 Pikachu."

We want to be able to do this in our controller:

```ruby
class BackpackPokemonsController < ApplicationController
  include ActionView::Helpers::TextHelper
  def create
    pokemon = Pokemon.find(params[:pokemon_id])
    @backpack.add_pokemon(pokemon.id)
    session[:backpack] = @backpack.contents
    flash[:notice] = "You now have #{pluralize(@backpack.count_of(pokemon.id), pokemon.name)}."
    redirect_to root_path
  end
end
```

Which will count the number of that specific pokemon in our backpack. When we run the test, we get an undefined method `count_of`. Let's write a model test first:

```ruby
class BackpackTest < ActiveSupport::TestCase
  test "has initial contents" do
    backpack = Backpack.new({ "1" => 1 })

    assert_equal({ "1" => 1 }, backpack.contents)
  end

  test "can add a pokemon" do
    backpack = Backpack.new({ "1" => 1})

    backpack.add_pokemon(1)
    backpack.add_pokemon(2)

    assert_equal({ "1" => 2, "2" => 1}, backpack.contents)
  end

  test "returns total number of all captured pokemon" do
    backpack = Backpack.new({ "1" => 3, "2" => 1, "3" => 3})

    assert_equal 7, backpack.total
  end

  test "returns total number of a specific pokemon" do
    backpack = Backpack.new({ "1" => 3, "2" => 1, "3" => 3})

    assert_equal 3, backpack.count_of(3)
  end
end
```

Run it. Same problem. Let's implement this `count_of` method:

```ruby
class Backpack
  attr_reader :contents
  def initialize(initial_contents)
    @contents = initial_contents || {}
  end

  def add_pokemon(pokemon_id)
    contents[pokemon_id.to_s] ||= 0
    contents[pokemon_id.to_s] += 1
  end

  def total
    contents.values.sum
  end

  def count_of(pokemon_id)
    contents[pokemon_id.to_s]
  end
end
```

Yay! All tests are passing. 

## You want more?

#### Showing the cart

Let's say that you wanted users to be able to click on "View Backpack" (similar to "View Cart" on an e-commerce site). 

* Which controller? 
* What does the view look like?
* How can we make the backpack have access to BackpackPokemon objects instead of just iterating through a hash of keys and values? 

#### Ending and saving the game

Now let's allow users to end their games.

```erb
  Backpack: <%= @backpack.count_all %>
  <%= flash[:notice] %>
  <%= button_to "End Game", games_path %>
```

In your routes:

```ruby
  resources :games, only: [:create]
```

Make a Games Controller:

```ruby
class GamesController < ApplicationController
  include ActionView::Helpers::TextHelper
  def create
    # the five lines below probably would be best delegated to a GameCreator PORO
    game = Game.new(user_name: "Rachel") do |game|
      @backpack.contents.each do |pokemon_id, quantity|
        game.game_pokemons.new(pokemon_id: pokemon_id, quantity: quantity)
      end
    end

    if game.save
      session[:backpack] = nil
      flash[:notice] = "Your game is finished! You captured #{game.pokemons.count} species of pokemon."
      redirect_to root_path
    else
      # implement if you have validations
    end
  end
end
```

## Other Resources
