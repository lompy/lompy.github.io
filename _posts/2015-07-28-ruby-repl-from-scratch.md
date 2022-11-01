---
layout: post
title:  "Ruby REPL from scratch"
categories: ruby repl from-scratch closure binding ripper
---
## Shortest REPL possible.

Ruby ships with default REPL (Read Eavaluate Print Loop) known as IRB
(Interactive RuBy) and every ruby programmer knows that program. I like to know
how stuff works so I've decided to write my own minimal viable REPL from
scratch.

And thanks to ruby's metaprogramming `eval` feature you can do it in 1 line
of code.

    # repl.rb
    # Loop Print      Eval Read
    loop { puts "=> #{eval(gets).inspect}" }

It's pretty darn minimal but let's see if it's viable.

Simple ruby one-liners.

    $ ruby repl.rb
    10 + 20
    => 30
    [1,2] - [1]
    => [2]
    {foo: 'bar', bar: 'foo'}.size
    => 2
    exit

Good. Let's try to remember some results.

    a = 10 + 20
    => 30
    a
    repl.rb:3:in `eval': undefined local variable or method `a' for main:Object (NameError)
      from min_repl.rb:3:in `eval'
      from min_repl.rb:3:in `block in <main>'
      from min_repl.rb:3:in `loop'
      from min_repl.rb:3:in `<main>'

In our viable REPL we want variables to persist between queries. Why don't they?

## Environment.
Turned out when I'm calling 'eval' in a loop every single call creates new
environment. The environment being a table of available variables with current
lexical scope and current `self` value. In ruby docs they call it
"execution context".  Here I'll use "environment" as it is shorter.

Environments can be nested. In that case variables from enclosing environment
are visible from child environment and can be affected in this environment.
We can create nested environment in ruby by crating a block.

    a = 10
    increment = proc { a += 1 }
    a #=> 10
    2.times(&increment)
    a #=> 12

But if we create the new environment first, then it doesn't know about `a`
variable created later in the enclosing environment.

    increment = proc { a += 1 }
    a = 10
    2.times(&increment)
    NoMethodError: undefined method `+' for nil:NilClass
      from (irb):2:in `block in irb_binding'
      from (irb):4:in `times'
      from (irb):4
      from /home/lompy/.rvm/rubies/ruby-2.2.1/bin/irb:11:in `<main>'

Call to `eval` basically works the same way as creating blocks regarding to
code being evaluated. Lets create a memory variable in enclosing environment
and see what happens.

    # repl.rb
    memory = {}
    loop { puts "=> #{eval(gets).inspect}" }

And in a REPL itself.

    $ ruby repl.rb
    memory[:foo] = 'bar'
    => bar
    memory
    => {:foo=>"bar"}
    exit

Cool, now we can persist some data. But can we do better?

## Binding.
Turned out we can as ruby provides another powerful metaprogramming feature --
`Binding`. An instance of `Binding` class holds a reference to environment
in which it was created and can be used later on with `eval` to evaluate
the code in environment referred by that binding. We can create binding object
anywhere with `Kernel#binding` and evaluate the code in the binding's
environment with `Binding#eval` or `eval(<code>, <binding>)`.

Let's use this knowledge to evaluate REPL code in the enclosing environment.

    # repl.rb
    b = binding
    loop { puts "=> #{b.eval(gets).inspect}" }

    $ ruby repl.rb
    a = 10
    => 10
    a
    => 10
    exit

Supercool, now we have variables. There also a constant
`Object::TOPLEVEL_BINDING` which refers top level execution environment as you
can conclude from name. So we can still hold on to one line of code.

    # repl.rb
    loop { puts "=> #{TOPLEVEL_BINDING.eval(gets).inspect}" }

Is it viable now? Not quite yet. Suppose we want to define a new class and we
want it to be readable by separating definition with line breaks.

    class Foo
    min_repl.rb:2:in `eval': <main>: syntax error, unexpected end-of-input (SyntaxError)
      from min_repl.rb:2:in `block in <main>'
      from min_repl.rb:2:in `loop'
      from min_repl.rb:2:in `<main>'

Doesn't work. We evaluate every single line hence every single line must be
valid ruby expression. If we want to separate expression we also need a read
loop. And in order to do that we need to come up with some kind of
a `valid_expression?` predicate to terminate the read loop. How can we check if
a string of ruby code is valid expression?

## Ripper.
Enter `Ripper`.

Of course we can rescue from `SyntaxError`s but there's better solution. The
thing is ruby standard library provides a ruby parser. How cool is that!
The name of the module is `Ripper`. Ripper can tokenize any string of ruby
and create abstract syntax trees with `Ripper.sexp(<code>)`. And if you pass
an invalid ruby expression to `sexp` it just returns `nil`. Let's get strait to
business.

    # repl.rb
    require 'ripper'

    def read
      print '> '
      result = ''
      loop do
        result += "#{gets}\n"
        break result if Ripper.sexp(result)
        print '* '
      end
    end

    loop { puts "=> #{TOPLEVEL_BINDING.eval(read).inspect}" }

I also added a `'* '` indicator of active read loop. Little test.

    $ ruby repl.rb
    > class Foo
    *   def bar
    *     'barbar'
    *   end
    * end
    => :bar
    > Foo.new.bar
    => "barbar"
    > exit

And this is just enough for me to be viable in scope of this post.

We have minimal viable REPL in ruby in less than 15 lines of code.
Pretty impressive!

## Beyond.
