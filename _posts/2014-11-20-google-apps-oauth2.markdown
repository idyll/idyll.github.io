---
layout: post
title:  "Google Apps Oauth2"
subtitle:  "Authenticating YOUR Users"
date:   2014-11-20 22:29:28
categories: ruby rails oauth
author: idyll
---
Google has deprecated their openid provider for Google Apps for Business. Super annoying, because the only option is to move over to their OAuth2 Google+ stuff. Here are the steps to get it going.

### Gemfile ###

First you need to get the requires gems into your Gemfile. In my case I am using devise with Rails 4.2beta so:

{% highlight ruby %}
gem 'devise', github: 'plataformatec/devise'
gem 'omniauth'
gem "omniauth-google-oauth2"
{% endhighlight %}

We need to get omniauth going so add a route:


{% highlight ruby %}
# /config/routes.rb
# ...

  devise_for :users, controllers: {
    omniauth_callbacks: "users/omniauth_callbacks"
  }
{% endhighlight %}

Then you need to setup a the callback controller you specified above:

{% highlight ruby %}
# /app/controllers/users/omniauth_callbacks_controller.rb
# ...

class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
  skip_before_filter :verify_authenticity_token
  include Devise::Controllers::Rememberable
  def index
    flash[:error] = "Access denied. Please sign in with a Google Apps account from #{ENV['GOOGLE_APPS_DOMAIN']}."

    redirect_to root_url
  end

  def google_oauth2
    # You need to implement the method below in your model (e.g. app/models/user.rb)
    auth_info = request.env["omniauth.auth"]

    # This allows you to pin login to a specific google apps domain.
    # In our case we wanted to restrict to our google apps domain.
    unless auth_info && auth_info['extra']['raw_info']['hd'] == ENV['GAPPS_DOMAIN']
      flash[:error] = "Access denied. Please sign in with a Google Apps account from #{ENV['GAPPS_DOMAIN']}."
      redirect_to root_url
    else

      @user = User.find_for_google_oauth2(auth_info, current_user)

      if @user.persisted?
        flash[:notice] = I18n.t "devise.omniauth_callbacks.success", :kind => "Google"
        sign_in_and_redirect @user, :event => :authentication
      else
        session["devise.google_data"] = auth_info
        redirect_to new_user_registration_url
      end
    end
  end
end
{% endhighlight %}

Then we need to add the method ```User.find_for_google_oauth2``` to our user model:


{% highlight ruby %}
# /app/models/user.rb
# ...
  def self.find_for_google_oauth2(auth, signed_in_resource=nil)
    user = User.where(:email => auth.info["email"]).first
    unless user
      user = User.new( provider:auth.provider, uid:auth.uid, name: auth.info.name,
        username: auth.info.email, email: auth.info.email, password: SecureRandom.base64(24) )
      # if your setup for nil passwords, you can also leave the password nil.
      user.save
    end
    user
  end
{% endhighlight %}

The last step is to configure devise. First lets add the oauth 2 provider to our devise initializer:

{% highlight ruby %}
# /config/initializers/devise.rb
# ...

  config.omniauth :google_oauth2, ENV["GAPPS_ID"], ENV["GAPPS_SECRET"], {
    prompt: 'select_account',
    scope: "email,profile"
  }
{% endhighlight %}


You will notice we specify ```select_account``` -  this is required because we need to prompt the user to pick which Google ID they will use if they are signed in with multiple ids.  
