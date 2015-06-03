# Vulcanize

Build simple form objects for custom value objects.

# NO CODE HERE
After spiking this general idea in another [project](https://github.com/CrowdHailer/scorched-blog/blob/master/lib/vulcanize.rb) I have decided to see what clarity can be brought to this project by defining the docs first. So have a read through and add whatever comments you want. Just remember this code doesn't do anything yet.
Few notes on Documentation Driven Design
- [one](http://tom.preston-werner.com/2010/08/23/readme-driven-development.html)
- [two](http://24ways.org/2010/documentation-driven-design-for-apis)

**[Pull request for comments](https://github.com/CrowdHailer/vulcanize/pull/1)**

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'vulcanize'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install vulcanize

## Usage

The first step is to create your value object. Your domain specific value object should implement the `typetanic/forge` protocol. The forge method can take any number of arguments to create a new value. It should also accept a block which is called with the error should creating the value object fail.

The forging of a value object should carry out all coercions and validations specific to that type. Vulcanize forms only know is a value is invalid or missing, not the reasons for a value to be invalid.

As an example here is a very simple implementation of a name object which has these conditions.
- It will always be capitalized
- It must be between 2 and 20 characters

```rb
class Name
  def initialize(raw)
    section = raw[/^\w{2,20}$/]
    raise ArgumentError, 'is not valid' unless section
    @value = section.capitalize
  end

  attr_reader :value

  def self.forge(raw)
    new raw
  rescue ArgumentError => err
    yield err
  end
end
```

Attributes in a form are defined with an attribute name and an attribute type. Often these will be the same but it may be the case that a form accepts attributes called first_name and last_name and where they both have the type name.
We can use our Name value object in a form as follows.

```rb
class Form < Vulcanize::Form
  attribute :name, Name
end

# Create a form with a valid name
form = Form.new :name => 'Peter'

form.valid?
# => true

form.errors
# => {}

form.values
# => {:name => #<Name:0x00000002a579e8 @value="Peter">}

# Create a form with an invalid name
form = Form.new :name => '<DANGER!!>'

form.valid?
# => false

form.validate!
# !! Vulcanize::InvalidForm

form.errors
# => {:name => #<ArgumentError: is not valid>}

form.values
# => {:name => '<DANGER!!>'}
```

### Null input
A types forge method is not called by default if the input given is nil.
To work with HTML forms empty string input is also treaded as a nil input.

Using the same form as above
```rb
form = Form.new :name => ''

form.valid?
# => true

form.errors
# => {}

form.values
# => {:name => nil}
```

### Default attribute
When declaring attributes a default may be provided for when the input is nil or empty.

```rb
class NullName
  def value
    'No name'
  end
end

class DefaultForm < Vulcanize::Form
  attribute :name, Name, :default => NullName.new
end

form = DefaultForm.new :name => ''

form.valid?
# => true

form.errors
# => {:name => #<Vulcanize::AttributeMissing: is not present>}

form.values
# => {:name =>nil}
```

### Required attribute
Perhaps instead of handling missing values you want the form to be invalid when values are missing. This can be set using the required option.

```rb
class RequiredForm < Vulcanize::Form
  attribute :name, Name, :required => true
end

form = RequiredForm.new :name => ''

form.valid?
# => false

form.values
# => {:name => nil}

form.name.value
# => 'No name'
```

### Private attribute
Sometimes input needs to be coerced or validated but should not be available outside the form. The classic example is password confirmation in a form.

```rb
class PasswordForm < Vulcanize::Form
  attribute :password, Password, :required => true
  attribute :password_confirmation, Password, :private => true

  def valid?
    unless password == password_confirmation
      error = ArgumentError.new 'does not match'
      errors.add(password_confirmation, error)
    end
    super
  end
end
```

### Renamed attribute
Vulcanize can also be used to handle any input that can be cast as a hash. JSON data for example. It may be that input fields need renaming. That can be done with the from attribute

```rb
class RenameForm < Vulcanize::Form
  attribute :name, Name, :from => 'display_name'
end

form = RenameForm.new :display_name => 'Peter'

form.valid?
# => true

form.values
# => {:name => #<Name:0x00000002a579e8 @value="Peter">}
```

### Checkboxes
A common requirement is handling checkboxes, there are two distinct patterns. The optional checkbox and the agreement checkbox.

##### Optional checkbox
This checkbox is true when checked and false when left unchecked

```rb
class OptionalForm < Vulcanize::Form
  attribute :recieve_mail, Vulcanize::Checkbox, :default => false
end

form = OptionalForm.new

form.valid?
# => true

form.values
# => {:recieve_mail => false}
```

##### Agreement checkbox
This checkbox must be selected to continue. The form should be invalid without its selection

```rb
class AgreementForm < Vulcanize::Form
  attribute :agree_to_terms, Vulcanize::Checkbox, :required => true
end

form = AgreementForm.new

form.valid?
# => false

form.errors
# => {:agree_to_terms => #<Vulcanize::AttributeMissing: is not present>}
```

##### Note on Vulcanize::Checkbox
*Vulcanize checkbox returns true for an input of 'on'. For all other values it raises an InvalidError instead of returning false. This is to help debug if fields are missnamed.*

## Standard types
Vulcanize encourages using custom domain objects over ruby primitives. it is often miss-guided to use the primitives. I.e. 12 June 2030 is not a valid D.O.B for your users and '<|X!#' is not valid article body. However sometimes it is appropriate or prudent to use the basic types and for that reason you can specify the following as types of attributes.

- String
- Integer
- Float

##### Note on using standard types
Often a reason to use standard types is because domain limitations on an input have not yet been defined. Instead of staying with strings consider using this minimal implementation.

```rb
class ArticleBody < String
  def self.forge(raw)
    new raw
  end
end
```
## TODO
- section on testing
- actual api docs, perhaps formatted as in [AllSystems](https://github.com/CrowdHailer/AllSystems#user-content-docs)
- Handling collections, not necessary if always using custom collections

## Questions
- Form object with required and default might make sense if default acts as null object?
- Form object should have overwritable conditions on empty
- Check out virtus Array and Hash they might need to be included in awesomeness
  - There is no need for and array or hash type if Always defining collections
  - general nesting structure checkout useuful music batch

## Contributing
There is no code here yet. So at the moment feel free to contribute around discussing these docs. pull request with your suggestion sounds perfect.

1. Fork it ( https://github.com/[my-github-username]/vulcanize/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request
