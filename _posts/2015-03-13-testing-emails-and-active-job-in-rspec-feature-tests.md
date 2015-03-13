---
layout: post
title: Testing Emails and ActiveJob#deliver_later in RSpec feature tests
categories: rpsec testing
---
I got tripped up by this the other day - I was writing feature specs around resetting passwords, and my `PasswordReset` interactor used `ActiveJob` to send the password reset email asynchronously.

Before `ActiveJob` you could just access `ActionMailer::Base.deliveries` and get at the message that had been generated, but that wasn't working. After a bit of searching and some failed attempts at other approaches, I figured out what needed to be done.

Since `ActiveJob` and `ActionMailer`'s `.deliver_later` queue up jobs to process later, you need to actually process the queue before you can access the email. To do this you need to first include `ActiveJob::TestHelper` in your feature spec. This gives access to a few methods, but the one that we need is `perform_enqueued_jobs`.

```
# /spec/features/password_reset_spec.rb
require "rails_helper"
include "ActiveJob::TestHelper"

feature "User Password Reset", type: :feature do
  before do
    user.save
  end

  let(:user){ FactoryGirl.build(:user, email: email, password: password) }
  let(:email){ "chris@example.com" }
  let(:password){ "passw0rd" }
  let(:new_password){ "passw0rd!!!" }

  scenario "User requests password reset" do
    visit "/forgot_password"
    fill_in "Email", with: email

    perform_enqueued_jobs do
      click_button "Reset Password"
    end

    expect(page)
      .to have_content(
        "Check your email for information on resetting your password")

    open_email(email)
    current_email.click_link "reset your password here"

    expect(page).to have_content("Set your password")

    fill_in "Password", with: new_password
    click_button "Set Password"

    expect(current_path).to eq "/user"
  end
end
```

`perform_enqueued_jobs` takes a block and performs any jobs in the `ActiveJob` queue after the block has completed. We pass the `click_button "Reset Password"` line in, and when that button is clicked the controller for password resets handles the process of generating the password reset email and putting it in to the queue. Without calling `perform_enqueued_jobs`, that's where it would sit, unprocessed and unavailable to our tests.

```
perform_enqueued_jobs do
  click_button "Reset Password"
end
```

Since the queue is processed, the email is now available for us to access. In this case I am using the [capybara-email](https://github.com/dockyard/capybara-email) gem to access the email and follow the link to it. From there everything continues as any normal feature spec would.

## Notes

One weird thing that I encountered while doing this was that while including `ActiveJob::TestHelper` at the top of the spec got it to pass, it broke a few other specs that were directly checking `ActiveJob::Base.queue_adapter.enqueued_jobs` started to break because `ActiveJob::Base.queue_adapter` was nil.

I haven't investigated this much so far, but moving `include "ActiveJob::TestHelper"` to `rails_helper.rb` fixed it, so I've left it at that for now.
