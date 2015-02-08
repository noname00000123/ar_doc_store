# ArDocStore

ArDocStore is a gem that makes it easy to handle document-store like behavior in ActiveRecord models that have access to the PostgreSQL JSON data type. You add a json column to your table called "data", include the ArDocStore::Model module, and then add schema-less attributes that get stored in the data column. With Ransack, these attributes are searchable as if they were real columns. 

There is also support for embedding models within models. These embedded models can be accessed from Rails form builders using fields_for.

The use case is primarily when you have a rapidly evolving schema with scores of attributes, where you would like to use composition but don't want to have a bajillion tables for data that fits squarely under the umbrella of its parent entity. For example, a building has entrances and restrooms, and the entrance and restroom each have a door and a route to the door. You could have active record models for the doors, routes, restrooms, and entrances, but you know that you only ever need to access the bathroom door from within the context of the building's bathroom. You don't need to access all the doors as their own entities because their existence is entirely contingent upon the entity within which they are contained. ArDocStore makes this easy.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'ar_doc_store'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install ar_doc_store

## Usage

The first thing you need to do is create a migration that adds a column called "data" of type ":json" to the table you want to turn into a document:

```ruby
change_table :buildings do |t|
	t.json :data
end
```

Then in the model file:

```ruby
class Building < ActiveRecord::Base
	include ArDocStore::Model
end
```

### Attributes

Now there are several ways to play but all boil down to one method on the model:
```ruby
class Building < ActiveRecord::Base
	include ArDocStore::Model
	
	attribute :name, :string
	attribute :width, :float
	attribute :height, as: :float # the :as is optional but if you like the consistency with SimpleForm
	attribute :storeys, :integer
	attribute :finished, :boolean
	attribute :construction_type, :enumeration, values: %w{wood plaster mud brick}, multiple: true, strict: true
end
```

Now I can do some fun things:
```ruby
building = Building.new name: 'Big Building', storeys: 45, construction_type: %w{mud brick}, finished: true
building.width = 42.5
building.height = 32.259
building.finished? # => true
building.save
```

You noticed the enumeration type on the construction_type takes an array. That's because I specified multiple: true. Let's say you can't have a mud and a brick building at the same time. Let's take off the multiple: true and see how it behaves:

```ruby
class Building ...
	attribute :construction_type, :enumeration, values: %w{wood plaster mud brick}, multiple: true, strict: true
end
building = Building.new
building.construction_type = 'wood'
building.construction_type  # => 'wood'
```

The strict: true option makes the enumeration strict, meaning only the choices in the enumeration are allowed. Maybe that should be the default choice, but the project in which this took shape had most of the enumerations with an "other" field. We took care of it with this and a custom SimpleForm input.

### Embedding Models

Let's say that a building has a door. The building is not the only thing in our world that has a door, so we want to be able to embed doors in other models, such as skating rinks, comfort stations, and gyms. First let's make an embeddable model called Door:

```ruby
class Door
  include ArDocStore::EmbeddableModel
	
  enumerates :door_type, multiple: true, values: %w{single double french sliding push pull}
  attribute :open_handle,  as: :enumeration, multiple: true, values: %w{push pull plate knob handle}
  attribute :close_handle, as: :enumeration, multiple: true, values: %w{push pull plate knob handle}
  attribute :clear_distance, as: :integer
  attribute :opening_force, as: :integer
  attribute :clear_space, as: :integer
end
```

Now let's put a Door on a Building:

```ruby
class Building ...
  embeds_one :door
  embeds_one :secret_door, class_name: 'Door'
end
```

Now let's make a building with a door:
```ruby
building = Building.new
building.build_door
building.door.clear_distance = 30
building.door.opening_force = 20
building.door.open_handle = %w{pull knob}
building.build_secret_door
building.secret_door.clear_distance = 3
building.save
```

We probably have a form for the building, so here goes:
```ruby
# in the controller:
def new
	@building = Building.new
end

def resource_params
	params.require(:building).permit :name, :height, door_attributes: [:door_type]
end

# in the view, with a bonus plug for Slim templates:
= simple_form_for @building do |form|
	= form.input :name, as: :string
	= form.input :height, as: :float
	= form.object.ensure_door
	= form.fields_for :door do |door_form|
		= door_form.input :door_type, as: :check_boxes, collection: Door.door_type_choices
```

What's to see here? Notice that I was able to "ensure_door", which means that if there already is a door, it keeps that one, otherwise it builds a new door object. Also on the door_type input, notice the collection comes from a door_type_choices that came from the enumeration attribute. Also notice that the embedded model conforms to the API for accepts_nested_attributes, for both assignment and validation, only you don't have to specify it because the _attributes= method comes for free.

You can also embeds_many. It works the same way:

```ruby
class Room
	include ArDocStore::EmbeddableModel
	attribute :length, as: :float
	attribute :width, as: :float
	attribute :height, as: :float
	enumerates :light_switch_type, %w{flip knob switchplate clapper}
end

class Building ...
	embeds_many :rooms
	embeds_one :foyer, class_name: 'Room'
end

building = Building.new
building.build_room # a bit different from active record here, I with that has_many used the has_one API for build_association.
building.ensure_room # if there are no rooms, then add one to the rooms collection, otherwise do nothing
building.rooms << Room.new(width: 12, height: 18, length: 20, light_switch_type: 'flip')
building.save  # saves the room
```

### Searching Models

In my dreams, I could type: Building.where(rooms: { length: 20 }). If somebody can guide me toward that dream, please do. In the meantime, there is Ransack. When you call "attribute" on a Model (not yet an embedded model, sorry) and you've got the Ransack gem installed, then you will get a free custom ransacker. So you can still do this, and it's pretty awesome:

```ruby
Building.ransack name_cont: 'tall', height_lteq: 20
```

### Custom attribute types

ArDocStore comes with several basic attribute types: array, boolean, enumeration, float, integer, and string. The implementation and extension points are inspired by SimpleForm. You can either create a new attribute type or overwrite an existing one. Forewarned is forestalled, maybe: as with SimpleForm, the custom input system is way easier to use if you were the one who built it, and it's still a little raw. Let's start with the implementation of :integer :

```ruby
class IntegerAttribute < Base
  def conversion
    :to_i
  end

  def predicate
    'int'
  end
end
```

Note that the data is getting put into a JSON column, so we want to make sure we get it out in the form that we want it. So the conversion method makes sure that it doesn't go in a integer and come back as a string. The predicate method tells postgres (via the ransacker) how to cast the JSON for searching.

Not all attribute types are that simple. Sometimes we have to put all the juice in the build method, and take care to define the getter and setter ourselves. In this example, we want to replace the boolean attribute with a similar boolean that can do Yes, No, and N/A, where N/A isn't simply nil but something the user has to choose. Here goes:

```ruby
class BooleanAttribute < ArDocStore::AttributeTypes::Base
  def build
    key = attribute.to_sym
    model.class_eval do
      store_accessor :data, key
      define_method "#{key}?".to_sym, -> { key == 1 }
      define_method "#{key}=".to_sym, -> (value) {
        res = nil
        res = 1 if value == 'true' || value == true || value == '1' || value == 1
        res = 0 if value == 'false' || value == false || value == '0' || value == 0
        res = -1 if value == '-1'
        write_store_attribute(:data, key, value)
      }
      add_ransacker(key, 'bool')
    end
  end
end

ArDocStore.mappings.merge! boolean: '::BooleanAttribute'

class BooleanInput < SimpleForm::Inputs::CollectionRadioButtonsInput
  def self.boolean_collection
    i18n_cache :boolean_collection do
      [ [I18n.t(:"simple_form.yes", default: 'Yes'), 1],
        [I18n.t(:"simple_form.no", default: 'No'), 0],
        [I18n.t(:"simple_form.na", default: 'N/A'), -1]]
    end
  end

  def input_type
    :radio_buttons
  end

end
```

## Roadmap
1. Default values for attributes. (I haven't needed yet...)
2. Ransackers for embedded model attributes. (I haven't needed yet...)
3. Refactor the EmbedsOne and EmbedsMany modules to use a Builder class instead of procedural metaprogramming. (Hello Code Climate...)
4. Currently when you mass-assign values to an embedded model, you need to assign all the values. It basically replaces what is there with what you send in, removing what has :_destroy set. I would be nice if it could do a smarter find or create behavior.


## Contributing

1. Fork it ( https://github.com/dfurber/ar_doc_store/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request