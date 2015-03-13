---
layout: post
title: DRYing Interactors with Modules and Shared Examples, Part 2
categories: interactors testing
---

In the [first post on DRYing interactors](/interactors/testing/2015/03/13/drying-interactors-part-1.html) we looked at creating modules and shared examples to simplify the process of checking the variables passed to interactors. Now we'll take things further and add some more complex functionality to a module by handling the process of creating billable items. In addition to adding methods to call from the interactors we'll also hook in a before block and create a rollback method from the module.

In the app I first wrote this for we have several interactors that calculate billing information on a variety of metrics. Initially all calculations had been done in a single interactor, but the file quickly got too long and did too much. Interactors should do a single unit of work, and the more you follow this rule the happier you'll be in the long run.

## Adding Billable Items

The core of the module we're setting up is to add a billable item to an array. Later we'll pass this array to another interactor, which will handle the process of charging for these items. That process will depend on the billing provider you're using, so we won't get in to that in this post.

{% highlight ruby %}
# app/interactors/calculate_usage.rb
class CalculateUsage
  include Interactor
  include BillableItems

  def call
    usage = calcuate_usage

    add_billable_item(
      description: "#{usage} Used",
      amount_in_cents: usage_charge_in_cents(usage),
      type: :minutes_used
    )
  end

private

  def calculate_usage
    ...
  end

  def usage_charge_in_cents(usage)
    ...
  end
end
{% endhighlight %}

This interactor calculates usage based on variables that we're not worried about today. What we're looking at is how `add_billable_item` provides a common interface for tracking what we need to bill a customer for as calculated across multiple interactors with our application.

`add_billable_item` is defined in the `BillableItems` module which is included near the top of the interactor. I like to keep files like this that are used across multiple interactors in `/app/interactors/concerns`. As mentioned in [part 1](/interactors/testing/2015/03/13/drying-interactors-part-1.html), we need to tell our application to load these files by adding

`config.autoload_paths += ["#{Rails.root}/app/interactors/concerns"]`

to `application.rb` if we're building a Rails app.

{% highlight ruby %}
# app/interactors/concerns/billable_items.rb
module BillableItems

private
  def add_billable_item(description:, amount_in_cents: 0, type: :general)
    context.billable_items << {
      description: description,
      amount_in_cents: amount_in_cents.to_i,
      type: type.to_sym
    }
  end
end
{% endhighlight %}

Right now the module just includes a single private method, `add_billable_item`, that adds a hash to an array. It's pretty straightforward, but right now it will not run successfully unless we set `context.billable_items` to an array before trying to add a billable item. We don't want to have to add this in every interactor that creates a billable item, so instead we'll set it up in the BillableItems module.

## Setting Up The Context in a before block

Interactors allow us to prepare things prior to running in a `before` hook. This is a good place to set up things like the billable items array we want to add our charges to, but in order to do so from our module we have to approach things a little differently than when we add a method.

{% highlight ruby %}
# app/interactors/concerns/billable_items.rb
module BillableItems
  def self.included(base)
    base.class_eval do
      before do
        context.billable_items ||= []
      end
    end
  end

private
  ...
end
{% endhighlight %}

This new method, `self.included`, is run when `BillableItems` is included in a class or module. We then pass in the class that it was included to and run `class_eval` on it, which lets us dynamically insert the `before` hook that we want in to the class. This allows us to ensure that the context includes a `bilable_items` array whenever the `BillableItems` module is included in any interactor.

## Rollback

Interactor rollback is called whenever an organized interactor fails, which gives the opportunity to undo any changes that were made. With our billing scenario we want to remove any billable items that were added by the interactor. We could explicitly add these to each interactor, but of course it'd be ideal to handle them within the `BillableItems` module instead.

First we need to know what the `billable_items` array looked like before we added anything to it. The easiest time to set this up is in the `before` hook that we just set up.

{% highlight ruby %}
# app/interactors/concerns/billable_items.rb
module BillableItems
  def self.included(base)
    base.class_eval do
      before do
        context.billable_items ||= []
        context.original_billable_items ||= []
        context.original_billable_items << context.billable_items.dup
      end
    end
  end

private
  ...
end
{% endhighlight %}

To start off we make sure that there's an `original_billable_items` array on the context. This will store the history of the `billable_items` array as we move through the organizer. Once we know the array exists we take the current `billable_items` array and duplicate it in to `original_billable_items`. When we rollback later we'll just take items out of the array as necessary.

Now that we have our history in place we can define the `rollback` method.

{% highlight ruby %}
# app/interactors/concerns/billable_items.rb
module BillableItems
  def self.included(base)
    ...
  end

  def rollback
    context.billable_items = context.original_billable_items.pop
    super
  end

private
  ...
end
{% endhighlight %}

`rollback` starts off by replacing the current billable_items with the last element of `original_billable_items` and removing it from the array. Then we call `super` in case the interactor itself has a `rollback` method that needs to perform any additional cleanup.

## The Full BillableItems Module

We've only looked at parts of the module as we've stepped through the pieces, so here's what the whole thing looks like:

{% highlight ruby %}
# app/interactors/concerns/billable_items.rb
module BillableItems
  def self.included(base)
    base.class_eval do
      before do
        context.billable_items ||= []
        context.original_billable_items ||= []
        context.original_billable_items << context.billable_items.dup
      end
    end
  end

  def rollback
    context.billable_items = context.original_billable_items.pop
    super
  end

private

  def add_billable_item(description:, amount_in_cents: 0, type: :general)
    context.billable_items << {
      description: description,
      amount_in_cents: amount_in_cents.to_i,
      type: type.to_sym
    }
  end
end
{% endhighlight %}

In my real world usage I also have methods to calculate the total charges contained in the `billable_items` array, which are run in `rollback` and `add_billable_items`. This simplifies the process of getting the total charges and keeps the logic around billable items all in one location.

## Tests

Last but not least (and they probably should've been first) are the tests. As in [part 1](/interactors/testing/2015/03/13/drying-interactors-part-1.html), RSpec's shared examples play a big part in keeping tests clean and repeatable.

{% highlight ruby %}
# spec/interactors/calculate_usage_spec.rb
require "rails_helper"

describe CalculateUsage do
  subject) do
    CalculateUsage.call(
      account: account,
      billable_items: billable_items
    )
  end

  let(:billable_items){ [] }
  let(:account){ double("Account", minutes_used: minutes_used) }
  let(:minutes_used){ 0 }

  describe ".call" do
    context "with all params" do
      let(:minutes_used){ 5 }

      it_behaves_like(
        "it has billable items of :type totaling :amount_in_cents",
        :minutes_used,
        20
      )

      it_behaves_like(
        "it has billable items of :type matching :description",
        :minutes_used,
        /5 Used/i
      )
    end
  end
end
{% endhighlight %}

By using shared examples we have an easily reusable set of tests that confirm that our calculations and billable item descriptions are being generated as expected.

{% highlight ruby %}
# spec/support/shared_examples_for_interactors_with_billable_items.rb
RSpec.shared_examples "it has billable items of :type totaling :amount_in_cents" do |type, amount|
  it "has billable items of #{type} totaling #{amount}"do
    expect(
      subject.billable_items.find_all{ |x| x[:type] == type }.sum{ |x| x[:amount_in_cents] }
    ).to eq(amount)
  end
end

RSpec.shared_examples "it has billable items of :type matching :description" do |type, description|
  it "has billable items of #{type} matching #{description}" do
    expect(
      subject.billable_items.find_all{ |x| x[:type] == type }.map{ |x| x[:description] }
    ).to include(description)
  end
end
{% endhighlight %}

Our shared examples here vary from what we used in [part 1](/interactors/testing/2015/03/13/drying-interactors-part-1.html) because they are receiving variables that we're using to populate the tests. This allows us the flexibility of testing for the actual values we expect, without requiring every interactor that involves billing calculations to get in to the nitty gritty of how a `billable_item` is structured.

Since we're also managing part of the rollback process from the module, we should also test that with a shared example.

{% highlight ruby %}
# spec/support/shared_examples_for_interactors_with_billable_items.rb
RSpec.shared_examples "billable items are rolled back" do
  it "includes the BillableItems module" do
    expect(described_class.included_modules.include?(BillableItems))
      .to be true
  end

  context "billable items are rolled back" do
    before{ interactor_parameters ||= {} }

    let(:interactor)do
      described_class.new(
        {
          billable_items: billable_items,
          original_billable_items: original_billable_items
        }.merge(
          interactor_parameters
        )
      )
    end

    let(:billable_items){ [{ description: "x", amount_in_cents: 200 }] }
    let(:original_billable_items) do
      [[{ description: "x", amount_in_cents: 300 }]]
    end

    let(:context){ interactor.context }

    it "adds a billable item" do
      expect{ interactor.call }.to change{ context.billable_items.size }.by(1)
    end

    it "reverts to the original billable items" do
      interactor.call
      expect{ interactor.rollback }.to(
        change{ context.billable_items }.to(original_billable_items.first)
      )
    end
  end
end
{% endhighlight %}

In addition to making sure that the `BillableItems` module is included in the class, we test that rollback works as expected. Since our tests don't run the interactors in an organizer we need to build up the `original_billable_items` array by hand, then make sure that adding a new billable item grows the history array, and srhinks it back down after rollback.

## Wrapping Up
A big part of the power of interactors is their ability to simplify and compartmentalize business logic in our applications. Moving common functionality, and the tests that deal with that functionality, out in to modules and shared examples helps us keep interactors simple, consistent, and understandable, without sacrificing capability.
