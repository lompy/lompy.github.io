---
layout: post
title:  "Ruby REPL from scratch"
categories: ruby repl from-scratch closure binding ripper
---
## Shortest REPL possible.

Ruby ships with default REPL (Read Evaluate Print Loop) known as IRB
(Interactive RuBy) and every ruby programmer knows that program. Let's write
our own REPL from scratch and learn something new in the process. We'll stick
with minimal viable version just to understand the very basics of building a
REPL.

Thanks to ruby's metaprogramming `Kernel#eval` feature you can do it in one line
of code.

    # repl.rb
    # loop print      eval read
    loop { puts "=> #{eval(gets).inspect}" }

We read user input with `gets` until encounter any new line character, then
evaulate aquired string with `eval` and print the result with `puts`. This is
pretty minimal but let's see if the first version is viable.

Simple ruby one-liners.

    $ ruby repl.rb
    10 + 20
    => 30
    [1,2] - [1]
    => [2]
    {foo: 'bar', bar: 'foo'}.size
    => 2
    exit

Good. Now let's try to remember some results between evaluations.

    $ ruby repl.rb
    a = 10 + 20
    => 30
    a
    (eval):1:in `block in <main>': undefined local variable or method `a' for main:Object (NameError)
            from repl.rb:3:in `eval'
            from repl.rb:3:in `block in <main>'
            from repl.rb:3:in `loop'
            from repl.rb:3:in `<main>'

In our minimal viable REPL we want variables to persist. Why don't they?

## Execution Context.
Turns out when I'm calling `eval` in a loop every single call creates a new
execution context. This is a table of all available variables within the
current lexical scope and the current `self` value.

Contexts can be nested. In that case variables from the enclosing context are
visible from the child context and can be affected in this context.
We can create nested context in ruby by crating a block.

    a = 10
    increment = proc { a += 1 }
    a #=> 10
    2.times(&increment)
    a #=> 12

But if we change the order of lines and create the proc first, then it doesn't
know about `a` variable created later in the enclosing context.

    increment = proc { a += 1 }
    a = 10
    2.times(&increment)
    (irb):1:in `block in <main>': undefined method `+' for nil:NilClass (NoMethodError)
            from (irb):3:in `times'
            from (irb):3:in `<main>'
            from /usr/lib/ruby/gems/3.0.0/gems/irb-1.3.5/exe/irb:11:in `<top (required)>'
            from /usr/bin/irb:23:in `load'
            from /usr/bin/irb:23:in `<main>'

Call to `eval` basically works the same way as creating blocks regarding to code
being evaluated. Lets create a memory variable in enclosing context and see what
happens.

    # repl.rb
    memory = {}
    loop { puts "=> #{eval(gets).inspect}" }

And in a REPL itself.

    $ ruby repl.rb
    memory[:foo] = "bar"
    => "bar"
    memory
    => {:foo=>"bar"}
    exit

Cool, now we can persist some data. But can we do better?

## Binding.
If IRB can then of cource we can do it too. Ruby provides another powerful
metaprogramming feature -- `Binding`. An instance of `Binding` class holds a
reference to the context in which it was created and can be used later with
`eval` to evaluate the code in the context referred by that binding. We can
create binding object anywhere with `Kernel#binding` and evaluate the code in
binding's context with `Binding#eval` or `Kernel#eval(<code>, <binding>)`.

Let's try to evaluate REPL code in the enclosing context.

    # repl.rb
    b = binding
    loop { puts "=> #{b.eval(gets).inspect}" }

    $ ruby repl.rb
    a = 10
    => 10
    a
    => 10
    exit

Supercool, now we have variables. The problem with this approach is that we are
not allowed to use variable `b` in our REPL.

    $ ruby repl.rb
    b = 10
    => 10
    b + 20
    repl.rb:3:in `block in <main>': private method `eval' called for 10:Integer (NoMethodError)
            from repl.rb:3:in `loop'
            from repl.rb:3:in `<main>'

We can overcome this using some difficult to guess variable name but there is a
better way. Ruby defines constant `Object::TOPLEVEL_BINDING` which refers top
level execution context as you can conclude from the name. Thanks to that we can
still keep our program in one line of code.

    # repl.rb
    loop { puts "=> #{TOPLEVEL_BINDING.eval(gets).inspect}" }

Is it viable now? Not quite yet. Suppose we want to define a new class and we
want the definition to be separated with line breaks for readability.

    $ ruby repl.rb
    class Foo
    repl.rb:2:in `eval': (eval):1: syntax error, unexpected end-of-input, expecting `end' (SyntaxError)
            from repl.rb:2:in `block in <main>'
            from repl.rb:2:in `loop'
            from repl.rb:2:in `<main>'

This doesn't work. We evaluate every single line hence every single line must be
valid ruby expression. If we want to separate expression with line breaks we also
need a read loop. And in order to do that we need to come up with some kind of
a `valid_expression?` predicate to terminate the read loop. How can we check if
a string of ruby code is valid expression?

## Ripper.
Of course we can rescue from `SyntaxError`s but there is a better solution. Ruby
standard library provides a ruby parser. How cool is that! The name of the module
is `Ripper`. Ripper can tokenize any string of ruby and create abstract syntax
tree with `Ripper.sexp(<code>)`. And if you pass an invalid ruby expression to
`sexp` it just returns `nil`. Let's see how can we implement the read loop.

    # repl.rb
    require 'ripper'

    def read
      print '> '
      result = ''
      loop do
        result += "#{gets}\n"
        # break from the loop on valid sexp and return result
        break result if Ripper.sexp(result)
        print '* '
      end
    end

    loop { puts "=> #{TOPLEVEL_BINDING.eval(read).inspect}" }

I added a `*` indicator of the active read loop. Let's do a little test.

    $ ruby repl.rb
    > class Test
    *   def initialize(str)
    *     @str = str
    *   end
    *   def hello
    *     puts "Hello, #{@str}"
    *   end
    * end
    => :hello
    > Test.new("reader").hello
    Hello, reader
    => nil
    > 

And this is just enough for me to be viable. We have minimal viable REPL in ruby
in less than 15 lines of code. Pretty impressive!

## More.
To make our REPL look more like IRB we can add things such as exception handling
or auto indentation and syntax highlighting, but that is outside of the scope of
this article.

Thank you for your time!
