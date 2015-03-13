One of my favorite developments from moving to [interactors](http://github.com/collectiveidea/interactor) in my Rails apps has been the way it's enabled me to clean up my controller tests. Before, my controller tests needed to know what combination of attributes would cause a failure when creating a new user, for example. If that changed, my tests broke.

Really, all a controller test should be looking at is what happens when the call is successful, and what happens when it fails. Do we load the page? Do we redirect? Do we display a flash message? That's what really matters, and interactors make it pretty easy to do that.

## Testing the Controller
We know that the controller is going to hit our `CreateUser` interactor, and whether or not it is successful determines what the user sees next. Rather than concerning ourselves with how to create success or failure we can instead stub `CreateUser.call` and only test what the controller is actually responsible for.

{% highlight ruby %}
# spec/controllers/users_controller_spec.rb
require "rails_helper"

RSpec.describe UsersController, type: :controller do
  describe "GET create" do
    before do
      allow(CreateUser).to receive(:call) do
        double("Interactor::Context", success?: success, user: user)
      end

      post :create, user: user_attributes
    end

    let(:user_attributes) do
      { email: "chris@example.com" }
    end

    let(:user){ double(User, name: "Chris") }

    context "service creation succeeds" do
      let(:success){ true }

      it{ expect(response).to be_redirect }
      it{ expect(flash[:notice]).to be_present }
    end

    context "service creation fails" do
      let(:success){ false }

      it{ expect(response).to render_template(:new) }
      it{ expect(flash[:alert]).to be_present }
    end
  end
end
{% endhighlight %}
An interactor returns an `Interactor::Context`, so we start by building a double that will represent that response, mocking any methods that we'd call on the context. That will always be `success?`, but could be anything else we might use to populate the view or determine what to do in the controller, like the user, account, or something else.

Since the `success` variable is what will determine what happens at the controller we'll set that up in with `let` inside each of the contexts. Then, when the test is run, the response from calling `CreateUser.call` will either be successful or not based on what we've set up at the top of the context.

## Setting up the Controller
Now that the tests are written we have a pretty good idea of what the controller needs to look like, and it's just as simple to understand as the tests.

{% highlight ruby %}
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  def create
    result = CreateUser.call(user_params: user_params)

    if result.success?
      redirect_to users_path, notice: "#{result.user.email} added."
    else
      flash[:alert] = "Unable to create user."
      @user = result.user
      render :new
    end
  end

private

  def user_params
    params.require(:user).permit(:email)
  end
end
{% endhighlight %}

Interactors let our controller do nothing but pass params to `CreateUser` and nothing else. The business logic is compartmentalized in the interactor (or in multiple organized interactors, we don't even need to know at the ctonroller level). We don't need to worry about validating the email, tracking any metrics, or doing anything else in the controller.

All that matters when deciding what to do next is checking if `result.success?` is true or not. The controller is easy to understand, and the tests are equally easy to implement.

## Wrapping Up
Obviously you could do something simliar by mocking `User.create` and returning a user double with `valid?` stubbed to true or false and achieve a similar result. I am not against this, but I belive that interactors are a superior approach to this, and to moving the manipulation of data to a single flow. I now do almost everything in interactors, even things that used to have in model callbacks.

When we decide to put our business logic in to interactors there are lots of benefits across the app. Already skinny controllers can get even skinnier, and their tests get even simpler to understand and create. The tests can stop worrying about how to make something fail and instead worry about what happens when it does.

