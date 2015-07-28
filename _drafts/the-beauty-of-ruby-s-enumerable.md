---
layout: post
title:  "The Beauty of Ruby's Enumerable"
categories: ruby functional-programming enumerable
---
## Intro.

Recently I got interested in functional programming, namely Clojure. And while
going through examples in some basic tutorials I really liked how collections
are handled in a unified manner via a bag of collection functions.

That got me thinking in how I treat collections in Ruby. Basic ones being
`Array` and `Hash`. And came to this ;tldr conclusions.

- Ruby has all the needs to handle collections in functional way.
- Ruby has beautiful syntax to chain collection method calls via blocks and dot
notation.
- Ruby has lazy collections since version 2.0.
- Collections are instances of classes with `Enumerable` module mixed in.
- Using `Enumerable#each` implies side effects. You rarely need them. Especially
you don't need `result = []` so much.
- Learn your standard library, it's awesomely expressive.

## Examples.

After learning about `Enumerable` I've decided to go through some production
code we are running. I found many examples of how using full power of
`Enumerable` can lead to short easy to understand constructions.

If you find yourself using constructions like

      result = []
      input_data.each { |d| result << d if d.good? }

you can do much better.


### 1. Symbol to proc with flat map.

Here is some code from our app.

      def locations_text
        result = []
        stages = object.stages.includes(:areas).includes(:landmarks)
        stages.each do |stage|
          if stage.areas.any?
            stage.areas.each { |area| result << area.title }
          else
            stage.landmarks.each { |landmark| result << landmark.title }
          end
        end
        text = result.join(" - ")
        text.blank? ? nil : text
      end

From reading through we conclude that we are collecting some titles of object's
stages' locations. We have three temporary variables `reault`, `stages` and
`text`, two conditionals.

Let's see if we can do better. First we are going to get rid of `result` and
`stages` by introducing some maps with symbol to proc arguments and a flat map
to unfold inner arrays.

      def locations_text
        text = object.stages
          .includes(:areas).includes(:landmarks)
          .flat_map { |s| s.areas.any? ?  s.areas.map(&:title) : s.landmarks.map(&:title) }
          .join(" - ")
        text.blank? ? nil : text
      end

Any better? Turned out Rails patches `Object` with `#presence` method to do
exactly what we do on last line. So we can ditch `text` variable as well.

      def locations_text
        object.stages
          .includes(:areas).includes(:landmarks)
          .flat_map { |s| s.areas.any? ?  s.areas.map(&:title) : s.landmarks.map(&:title) }
          .join(" - ").presence
      end

Here you go! Using our knowledge of `Enumerable` and core rails methods we twice
reduced lines count. More than that, now we reveal intension of the method in a
very strait manner without any variables. Just take stages, map them to areas'
titles or landmarks' titles, join titles and return `nil` if resulting string is
empty.

### 2. Each slice.
Somehow I bumped at this code in [elasticsearch-rails]
(https://github.com/elastic/elasticsearch-rails).

      def __find_in_batches(options={}, &block)
        options[:batch_size] ||= 1_000
        items = []

        all.no_timeout.each do |item|
          items << item

          if items.length % options[:batch_size] == 0
            yield items
            items = []
          end
        end

        unless items.empty?
          yield items
        end
      end

But having read through line by line I realized that it is just slicing of the
`all` array and yielding parts of it to the block. Given that I've read through
`Enumerable` docs the same day I immediately came up with the clean
[refactoring] (https://github.com/elastic/elasticsearch-rails/pull/378/files).

      def __find_in_batches(options={}, &block)
        options[:batch_size] ||= 1_000

        all.no_timeout.each_slice(options[:batch_size]) do |items|
          yield items
        end
      end

We got three lines instead of eleven. Eleven.

If we do not need side effects we can map 

### 3. Take it or drop it.

Have some collection and need to select some of them until some condition is
truthy? Four months ago I came up with something like this.

      def versions_since_publish
        result = []
        versions.each do |v|
          break if v.site_published_at_updated?
          unless v.search_indexed_at_updated?
            result << v
          end
        end
        result
      end

And compare with the clear collection-wise implementation.

      def versions_since_publish
        versions
          .take_while { |v| !v.site_published_at_updated? }
          .reject     { |v| v.search_indexed_at_updated? }
      end

### 4. Hash is unordered collection.

The default representation of hash entry is array of key-value pair:

      {:foo => "bar"}.entries
      # => [[:foo, "bar"]]

Also you can create hashes from entries

      Hash[ [[:foo, "bar"]] ]
      # => {:foo => "bar"}

So instead of this:

      def tour_hotels
        result = {}
        object.tour_hotels.each do |th|
          result[th.hotel_id] = th
        end
        return result
      end

We can do better:

      def tour_hotels
        Hash[ object.tour_hotels.map { |th| [th.hotel_id, th] } ]
      end

### 5. Divide and conquer.

What does this method do?

      class Area
        ...

        def belongs_to_or_owns?(area)
          result = true
          [:country, :region, :city, :distinct].each do |area_type|
            if self.respond_to?(area_type) && area.respond_to?(area_type)
              result = false if self.send(area_type) != area.send(area_type)
            end
          end
          result
        end

        ...
      end

Hard to say. Lets divide it into smaller methods and conquer the meaning:

      class Area
        ...

        def belongs_to_or_owns?(other)
          owns?(other) || other.owns?(self)
        end

        protected

        def owns?(other)
          parents_and_self_ids.include?(other.id)
        end

        def parents_and_self_ids
          [:distinct, :city, :region, :country]
            .select { |m| respond_to?(m) }
            .map { |m| send(m).id }
        end

        ...
      end
