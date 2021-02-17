# Introduction

**Note, I have spoken to Evan Phoenix who emailed me: "say that I indicated you should submit it."**

What makes Ruby succeed as a DSL is that it allows removing clutter away so that Ruby can read like good English. I didn't understand that when I first learned Ruby on Rails.

Many years ago, I came to Ruby via Michael Hartl's well-known Rails Tutorial. The Ruby syntax seemed familiar, but much of it seemed like magic that I cut and pasted and ran to see if it worked. I had only a vague idea of how the code "worked," even though I had spent the prior 15 years mastering Java.

It dawned on me that if my mind could think like the interpreter, then lines of Ruby code would go from magic to being mainly a few basic programming concepts, such as assigning values to variables and sending messages to objects.

In this talk, I plan to transform the code into uglier, more explicit syntax with "self," parentheses, periods, etc., so that what once seemed magical then seems ordinary. I'll keep running the unit tests to show the explicit changes are optional.

Most Ruby code boils down to

* Defining variables, including local variables and constants like modules and classes
* Sending a message (method name and arguments) to some object.
* Ruby classes are are also objects that take messages, so not just created objects based on calling `SomeClass.new`.

I'll cover the major differences between JavaScript and Ruby in terms of explicit vs. implicit.

## Parentheses

### JS: Parens mean function invocation
Parens are never optional. Parens signifying calling a function defined by the symbol. Otherwise, a symbol points to some value, which might be a function.

### Ruby: Implicit parentheses for method calls
Parentheses are almost always optional.

Example of "Magic Syntax" from a Rails model setup

```ruby
has_many :following, through: :active_relationships,  source: :followed
```
To make this more explicit, we can first add parentheses: 

```ruby
has_many(:following, { through: :active_relationships,  source: :followed})
```

In Ruby, Zero arg method calls are indistinguishable from local variables
  
Implicit:

```ruby
User.first
```

Explicit:

```ruby
User.first()
```

* Method calls without parentheses are hard to spot for those not used to the optionality of parentheses
* Operators in Ruby code boil down to calling a method on some object with arguments. We can add parentheses to operator calls.
  
Implicit parens and dot for an operator:

```ruby
def follow(other_user)
  following << other_user unless self == other_user
end
```

Explicit parens and dot for an operator:

```ruby
unless self == other_user
  self.following().<<(other_user) 
end
```

## Variable Declarations
### JS: Explicit declaration of variables
In JS, use `var`, `let`, or `const`
```js
var x = 0;
```

Note, if you make an assignment without those, you may end up creating a variable on the global object, `global` or `window`.
```js
x = 0;
```

is the same as
```js
window.x = 0;
```

### Ruby: Implicit local variable declaration

Local variable definitions are _implicit_ by assigning a value and those take precedence over calling a method on self. Otherwise, if the identifier is not a local variable, you're sending a message to the implicit self.

```ruby
def downcase_email
 self.email = email.downcase
end
```  

Which is really the method call:

```ruby
self.email=(email.downcase)
```

And not this because of the implicit variable local variable declaration of email

```ruby
def downcase_email
  email = email.downcase # bug b/c this defines a new local variable email
end
```

## Ruby's Implicit receiver of messages

### JS: No Implicit Receiver
JS has no implicit receivers to messages. JS's `this` is a bit like Ruby's `self` in that it can refer to the object containing the function. But `this` is _never_ implicit.

However, since JS uses scopes to figure out what some symbol refers to, it might seem that the `window` or `global` is is the implicit receiver of messages since that scope, being at the top level, is always available.
  
### Ruby: Implicit Receiver 
The implicit `self` is the receiver of a message if there is no other receiver specified.
    
Per the previous example of Rails model setup, `has_many` is simply a method call on self.

```ruby
self.has_many(:following, { through: :active_relationships,  source: :followed})
```
              
But what is `self`? Is it an instance of User? Nope!
   
One way to see the implicit `self` is to print it out, using amazing print.
                 
```ruby
  puts "self = #{self.ai}"
```

and that prints out:

```
self = class User < ApplicationRecord {
   <truncated>           
}
```

Another way to figure out how the magic of `has_many` works is to use the `pry` gem, especially with the `pry-doc` gem:

```
[3] [sample_app][development] pry(User)> show-source has_many

From: /Users/justin/.rvm/gems/ruby-2.6.3/gems/activerecord-6.1.0/lib/active_record/associations.rb:1229:
Owner: ActiveRecord::Associations::ClassMethods
Visibility: public
Signature: has_many(name, scope=?, ?, &extension)
Number of lines: 4

def has_many(name, scope = nil, **options, &extension)
  reflection = Builder::HasMany.build(self, name, scope, options, &extension)
  Reflection.add_reflection self, name, reflection
end
```

Tip, `show-doc` is similarly useful:

```
[2] [sample_app][development] pry(User)> show-doc has_many

Owner: ActiveRecord::Associations::ClassMethods
Signature: has_many(name, scope=?, ?, &extension)

Specifies a one-to-many association. The following methods for retrieval and query of
collections of associated objects will be added:
<lots more>
```

However, when passing around a block, the implicit `self` inside the block can either be from the place the block is defined or from the caller of the block if `instance_eval` or `instance_exec` is used. For example, a Rails `config` block uses this trick so that within the block, the code has access to the `config` object.

One big gotcha with the implicit `self` is that creating local variables takes precedence over using the implicit `self`.

Thus, on the left-hand side of an equal sign, Rails devs need to use `self.some_attribute =`, like `self.email =` something so that Ruby knows that some object should receive a message rather than make a new variable declaration.


## Returning values
### JS: explicit return
`return` must be explicit for all functions, other than the fat arrow, no braces type.

### Ruby: implicit return
Implicit return of the last statement value for blocks and methods.

## Lookup of Ruby Constants vs. JS Explicit Imports

### JS: Explicit 
In ES6, all identifiers are explicitly "imported" into a file.

```js
import React from 'react';
```

### Ruby
* Case matters. Uppercase signifies a constant that uses the rules for constant lookup.
* Lowercase is either a local variable or a message sent to some object, maybe the explicit self.
* Calling `ancestors` on a class shows the lookup rules for finding the method to be called. However, it's easier to use `pry` for this sort of introspection.
* Constants are never found on self.
* All Ruby constants are always accessible via specific lookup rules. Constants are available when files are loaded.

## Method Missing
                          
### JS
Js has no method missing.  

### Ruby
`method_missing` allows the creation of "magical" DSLs. A simple example is the Rails configuration object, where new methods are added by simply assigning a value.

Here's a little pry session that shows this, using `binding.pry` in `development.rb`

```
From: /rescript-on-rails-tutorial/config/environments/development.rb:9 :

     4:   # Settings specified here will take precedence over those in config/application.rb.
 =>  9:   config.cache_classes = false
    10:

[1] [sample_app][development] pry(#<SampleApp::Application>)> config.cache_classes
nil
```

Lets's put in an undefined config name, add `x` to end of the last line:

```
[2] [sample_app][development] pry(#<SampleApp::Application>)> config.cache_classesx
NoMethodError: undefined method `cache_classesx' for #<Rails::Application::Configuration:0x00007f8cf663dd70>
Did you mean?  cache_classes
               cache_classes=
from /Users/justin/.rvm/gems/ruby-2.6.3/gems/railties-6.1.0/lib/rails/railtie/configuration.rb:97:in `method_missing'
```

Now, let's set and retrieve a value on `config`.

```
[3] [sample_app][development] pry(#<SampleApp::Application>)> config.foobar = "x"
"x"
[4] [sample_app][development] pry(#<SampleApp::Application>)> config.foobar
"x"
```

The above _magic_ works via `method_missing`.
                                                           
Incidentally, `pry` is super useful to see the "api" of `config`

```
[6] [sample_app][development] pry(#<SampleApp::Application>)> show-source config

From: /Users/justin/.rvm/gems/ruby-2.6.3/gems/railties-6.1.0/lib/rails/application.rb:395:
Owner: Rails::Application
Visibility: public
Signature: config()
Number of lines: 3

def config #:nodoc:
  @config ||= Application::Configuration.new(self.class.find_root(self.class.called_from))
end
```


## Summary
There is no "magic" in the Rails DSL once you understand how you can put in a few print statements or do some exploration in `pry`. 





