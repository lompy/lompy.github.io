---
layout: post
title:  "From SQL joins to AR::Relation#{preload, indludes, eager_load, joins} and Arel"
date:   2015-04-13 22:00:42
categories: SQL rails active-record arel
---

- Ruby version: 2.2.1
- Rails version: 4.2

I allways forget how exactly SQL joins are working. Which is left join and which is right?

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
