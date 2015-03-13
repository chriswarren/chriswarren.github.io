---
layout: post
title: DRYing Interactors with Modules and Shared Examples, Part 1
categories: interactors testing
---

I've been using the [interactor gem](https://github.com/collectiveidea/interactor) in my Rails apps lately and I love the way they simplify complex logic and help clarify what my apps do. As I've worked with them I've developed a few testing tricks using modules and shared examples that have helped make the interactors and their tests cleaner and more consistent.

## Modules for Common Contexts

Interactors can be run individually or chained together using organizers. When using organizers it's important to make sure that the context items we're using are consistently named, otherwise we might end up with one interactor expecting `user` and another expecting `current_user`.

One way that I've handled this is by finding those common elements across many interactors and pulling checks for them out in to modules that can be included in all interactors that deal with those elements.

Consider the following interactor:

{% highlight ruby %}
# app/interactors/add_user_to_account.rb
class AddUserToAccount
  include Interactor

  def call
    context.fail!(message: "User required.") unless context.user.present?
    context.fail!(message: "Account required.") unless context.account.present?

    context.account.users << context.user
  end
end
{% endhighlight %}

While this is a fairly simple example, odds are good that we're going to start to see some common requirements across multiple interactors, especially when dealing with elements that are widely used in our applications. In this case, let's take the `user` requirement and move it to a module.

{% highlight ruby %}
# app/interactors/concerns/user_requirements.rb
module UserRequirements

private
  def require_user(message: "User required.")
    context.fail!(message: message) unless context.user.present?
  end
end
{% endhighlight %}

Pretty straight forward - we've copied the first check from our interactor in to the `require_user` method, and set it up so the message can be overwritten if needed.

Before we can use it in our interactors we'll want to make sure that our app knows to load these files. In a Rails app we'll add

`config.autoload_paths += ["#{Rails.root}/app/interactors/concerns"]`

to `application.rb`.

Now that it's loaded and available (don't forget to restart your server and console so the change to `application.rb` takes effect), we can use it in our interactor.

{% highlight ruby %}
# app/interactors/add_user_to_account.rb
class AddUserToAccount
  include Interactor
  include UserRequirements

  def call
    require_user
    context.fail!(message: "Account required.") unless context.account.present?
    context.account.users << context.user
  end
end
{% endhighlight %}

That's all it takes to start using the module and simplify the user requirement check in our interactors. We could do the same thing for Accounts easily enough. As we start to have other user requirements we can add them to the `user_requirements.rb` file as well. Some of the ones I usually end up with include `require_admin_user`, `require_paid_user`, and `require_account_owner`.

## Testing

I assume you already have some tests for your interactor and confirmed that these changes didn't break anything. Awesome. Now let's take some steps to simplify our interactor tests using shared examples, a feature of [RSpec](http://rspec.info).

Since we're moving our checks for users to a module to reduce repetition, it makes sense to do the same thing for our tests as well. Plus, with shared examples it's easy to be even a little more thorough since the tests are slightly hidden from the actual spec files.

Here's our starter test for the interactor we worked on earlier, which lives at `spec/interactors/add_user_to_account_spec.rb`.

{% highlight ruby %}
# spec/interactors/add_user_to_account_spec.rb
require "rails_helper"

describe AddUserToAccount do
  let(:interactor) do
    AddUserToAccount.call(
      account: account,
      user: user
    )
  end

  let(:account){ FactoryGirl.create(:account) }
  let(:user){ FactoryGirl.create(:user) }

  describe ".call" do
    context "without a user" do
      let(:user){ nil }
      it{ expect(interactor.failure?).to be true }
    end

    context "without an account" do
      let(:account){ nil }
      it{ expect(interactor.failure?).to be true }
    end

    it{ expect(interactor.success?).to(be true) }
    it{ expect{ interactor }.to change{ user.account }.to(account) }
  end
end
{% endhighlight %}

Now let's move the checks that make sure a user is present in the context to a shared concern.

{% highlight ruby %}
# spec/support/shared_examples_for_interactors_with_users.rb
RSpec.shared_examples "a user is required" do
  context "without a user" do
    let(:user){ nil }
    it{ expect(interactor.failure?).to be true }
  end
end
{% endhighlight %}

With the spec logic duplicated from `add_user_to_account_spec.rb`, we'll start by replacing that logic from the shared concern.

{% highlight ruby %}
# spec/interactors/add_user_to_account_spec.rb
require "rails_helper"

describe AddUserToAccount do
  let(:interactor) do
    AddUserToAccount.call(
      account: account,
      user: user
    )
  end

  let(:account){ FactoryGirl.create(:account) }
  let(:user){ FactoryGirl.create(:user) }

  describe ".call" do
    it_behaves_like "a user is required"

    context "without an account" do
      let(:account){ nil }
      it{ expect(interactor.failure?).to be true }
    end

    it{ expect(interactor.success?).to(be true) }
    it{ expect{ interactor }.to change{ user.account }.to(account) }
  end
end
{% endhighlight %}

Now, let's expand on the shared concern to make it more thorough. Since we want to make sure that any interactor that requires a user is using the `UserRequirements` module, we can easily add a check that it is being included. This is an easy way to enforce consistent usage of the file and naming conventions across interactors.

{% highlight ruby %}
# spec/support/shared_examples_for_interactors_with_users.rb
RSpec.shared_examples "UserRequirements is included" do
  it "has included UserRequirements" do
    expect(
      described_class.included_modules.include?(UserRequirements)
    ).to be true
  end
end

RSpec.shared_examples "a user is required" do
  it_behaves_like "UserRequirements is included"

  context "without a user" do
    let(:user){ nil }
    it{ expect(interactor.failure?).to be true }
  end
end
{% endhighlight %}

We don't need to actually to explicitly call the check for the inclusion of UserRequirements in our tests. Since we want to do the check whenever we're verifying that a user is required, we can do it within the shared example we wrote originally.


Later, if you add more methods to the `UserRequirements` module you can add related tests that build upon each other here.

## Wrapping Up

These aren't optimizations that need to be made immediately when using interactors, but I do use them as soon as I find myself requiring any common variables in more than one interactor, and especially when chaining interactors across organizers.

This same approach can be used for more than just requiring variables be present. In [part 2 of DRYing interactors with modules and shared examples](/interactors/testing/2015/03/13/drying-interactors-part-2.html) we'll explore a way of extracting common logic from interactors, like creating invoice items for a variety of items without duplicating logic.
