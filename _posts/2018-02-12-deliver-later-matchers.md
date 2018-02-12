---
layout: post
title: "Verifying Background Emails in Rails"
date: 2018-02-12 16:03:00 -0400
categories: testing gems
---

I was looking at one of [our](https://maketime.io) Rails apps to see how we
covered delayed mailer calls.  So, for example, we have an application with an
EmailCampaign model.  An EmailCampaign `has_many :recipients`.  Recipients
are essentially email addresses.

Okay, so, in typical Rails fashion, we have an `EmailCampaignMailer` and we will
use this mailer to send out emails to all of an EmailCampaign's recipients
after the EmailCampaign is created.  So our controller action, looks something
like this:

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
do not have the logic to send an email campaign in our controller.  I use the
[Interactor gem](https://github.com/collectiveidea/interactor) extremely heavily
in practice; however, to avoid introducing new concepts, I will keep this example
as straightforward as possible.

So, my question was, given that an EmailCampaign is created, how can I verify
that an email is sent to each of its recipients.  Furthermore, how can I verify
that the emails are sent in the background using `deliver_later`?
