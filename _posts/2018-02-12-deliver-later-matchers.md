---
layout: post
title: "Verifying Background Emails in Rails"
date: 2018-02-12 16:03:00 -0400
excerpt: I made gem to more easily test whether `ActionMailer` emails will be delivered later.
categories: testing gems
---

## TL;DR

I made gem to more easily test whether `ActionMailer` emails will be delivered
later.

If you want to use the `deliver_later_matchers` gem, you can find it on
[RubyGems](https://rubygems.org/gems/deliver_later_matchers)
and [GitHub](https://github.com/jdlubrano/deliver_later_matchers).

## Providing Some Context

I was looking at one of [our](https://maketime.io) Rails apps to see how we
covered delayed mailer calls.  So, for example, we have an application with an
EmailCampaign model.  An EmailCampaign `has_many :recipients`.  Recipients
are essentially email addresses.

In typical Rails fashion, we have an `EmailCampaignMailer` that we will use to
send out emails to all of an `EmailCampaign`'s recipients after the
`EmailCampaign` is created.  Our controller action for creating an
`EmailCampaign`, looks something like this:

```ruby
class EmailCampaignsController < ApplicationController
  def create
    @email_campaign = EmailCampaign.new(email_campaign_params)
    @email_campaign.recipients = recipient_params

    if @email_campaign.save
      send_email_campaign
      redirect_to :show, notice: 'Email Campaign will be sent'
    else
      render :new
    end
  end

  private

  def send_email_campaign
    @email_campaign.recipients.each do |recipient|
      EmailCampaignMailer.campaign_email(recipient).deliver_later
    end
  end
end
```

Now, before I upset everybody who prefers their controllers skinny, we actually
do not have the logic to send an `EmailCampaign` in our controller.  I use the
[Interactor gem](https://github.com/collectiveidea/interactor) extremely heavily
in practice; however, to avoid introducing new concepts, I will keep this example
as straightforward as possible.

So, my question was, given that an EmailCampaign is created, how can I verify
that an email is sent to each of its recipients.  Furthermore, how can I verify
that the emails are sent in the background using `deliver_later`?

## First Attempt (stubbing)

My first thought was to stub out `deliver_later` and verify that it was called;
however, that would require stubbing a chain of methods,
`EmailCampaignMailer#campaign_email` and `deliver_later`.
So a request spec for my create endpoint would look something like:

```ruby
it 'delivers a campaign email later to each recipient' do
  recipients = [{ email: 'test1@test.com' }, { email: 'test2@test.com' }]

  params = {
    body: 'This is my email body',
    subject: 'This is my email subject',
    recipients: recipients
  }

  mail_double = double(deliver_later: nil)
  allow(EmailCampaignMailer).to receive(:campaign_email).and_return(mail_double)

  post '/email_campaigns', params: params

  recipients.each do |recipient|
    expect(EmailCampaignMailer).to have_received(:campaign_email).with(recipient)
  end

  expect(mail_double).to have_received(:deliver_later).twice
end
```

Not terrible, certainly not the worst test I have written, but it feels like
the most complicated part of the test is setting up  and verifying the doubles
and stubs.  Especially if a less experienced developer were to read to this
test, the setup may be confusing.

## Maybe There is a Better Way

I started searching for a better solution, something that would hide some of the
test complexity, and I found an [issue](https://github.com/rspec/rspec-rails/issues/1901)
in the RSpec Rails repository where another developer, [@fabn](https://github.com/fabn),
had requested a new matcher for ActionMailer calls.  One of the maintainers
of RSpec Rails suggested that ActionMailer matchers provided an excellent
opportunity for creating an extension gem.

I thought, I can do that; I can write gems.  I started to look under the hood
of `ActionMailer` and at the RSpec [documentation](https://relishapp.com/rspec/rspec-expectations/v/3-7/docs/custom-matchers)
for writing custom matchers. Essentially, RSpec provides a really nice DSL for
creating one's own custom matchers, so that piece of the gem ended up being
pretty straightforward.  Perhaps the more interesting piece of the gem pertains
to how `deliver_later` works.

What @fabn and I ultimately wanted was something like:

```ruby
it 'delivers a campaign email later to each recipient' do
  recipients = [{ email: 'test1@test.com' }, { email: 'test2@test.com' }]

  params = {
    body: 'This is my email body',
    subject: 'This is my email subject',
    recipients: recipients
  }

  expect {
    post '/email_campaigns', params: params
  }.to enqueue_email(EmailCampaignMailer, :campaign_email).with(recipients.first)
    .and enqueue_email(EmailCampaignMailer, :campaign_email).with(recipients.last)
end
```

That test case seems to match up with the real-life scenario much more
clearly.  We expect that creating an `EmailCampaign` will send emails to each
recipient.  We will discuss how to achieve this below.

## Diving Into ActionMailer

Alright, so, a crash course on ActionMailer and delayed emails.  When you call
`deliver_later` on an `ActionMailer` email method

```ruby
EmailCampaignMailer.campaign_email(recipient).deliver_later
```

what happens?  Well, `ActionMailer` enqueues an `ActiveJob` to deliver the
email.  If you read the aforementioned
[GitHub issue](https://github.com/rspec/rspec-rails/issues/1901), you see that
`ActionMailer` enqueues an `ActionMailer::DeliveryJob` with the following
arguments:

```ruby
['<MailerClass>', '<email_method>', 'deliver_now', *other_args]
```

So for our `EmailCampaignMailer` example, an `ActionMailer::DeliveryJob` with

```ruby
['EmailCampaignMailer', 'campaign_email', 'deliver_now', recipient]
```

would be enqueued.

At this point, we know what we want to verify in our tests.  We want to verify
that an `ActionMailer::DeliveryJob` with a given set of arguments is enqueued.

## Verifying Changes with a Block

Recall that the desired syntax for `deliver_later_matchers` was:

```ruby
expect { EmailCampaignMailer.campaign_email(recipient).deliver_later }
  .to enqueue_email(EmailCampaign, :campaign_email)
  .with(recipient)
```

So the matcher needs to run the given block and verify that the block adds a
matching `ActionMailer::DeliveryJob` to the `ActiveJob` queue.  So, really, we
need to know which jobs are added by the block and then search through those
jobs to find one that matches our expected arguments.

It turns out that achieving this is not all that difficult.  In
`lib/deliver_later_matchers/deliver_later.rb`, we have the `matches?`
method (shortened for brevity here):

```ruby
def matches?(block)
  existing_jobs_count = enqueued_jobs.size
  block.call
  jobs_from_block = enqueued_jobs[existing_jobs_count..-1]

  matching_job_exists?(jobs_from_block)
end
```

`matching_job_exists?` runs through the jobs enqueued by the block
and determines whether an `ActionMailer::DeliveryJob` exists with the expected
arguments.  This is actually the same strategy employed by the
[ActiveJob Matchers](https://github.com/rspec/rspec-rails/blob/master/lib/rspec/rails/matchers/active_job.rb)
in RSpec Rails.  There may have been some way to incorporate the ActiveJob
matchers to avoid duplicating some logic, but I was worried that doing so
would couple my gem too tightly to RSpec Rails.

## Acknowlegdments

If you want to use the `deliver_later_matchers` gem, you can find it on
[RubyGems](https://rubygems.org/gems/deliver_later_matchers)
and [GitHub](https://github.com/jdlubrano/deliver_later_matchers).

Thanks to [@fabn](https://github.com/fabn) for creating the issue in RSpec Rails
and sending me down the right path with regards to `ActiveJob`.

Thanks to the folks at RSpec for awesome documentation.

Thanks to [Thoughtbot](https://thoughtbot.com) and their
[json_matchers](https://github.com/thoughtbot/json_matchers) gem from which
I borrowed the architecture for the `deliver_later_matchers` gem.
