# DeviseInvitable + Rails API + Ember

## Intro

In a recent project, I put together Rails API backend and a Ember CLI frontend for one of my awesome clients. With Rails5, [Devise](https://github.com/plataformatec/devise), and [Ember Simple Auth Devise](https://github.com/simplabs/ember-cli-simple-auth-devise) it's a beaut of a project; Being both a pleasure to work in as well as being an environment which is highly productive. One tiny hill I had to run up while working on this project was using the [DeviseInvitable](https://github.com/scambra/devise_invitable) (DV) plugin to create a email invite sign up flow for the app's users. DV adds some out of the box routes and a mailer to your application so you can simply call `User.invite!(email: 'mike@saulgoodman.com')` and it will create a user, assign them an invite token, and set them up so they can sign up via the email that it sends out.

In my project, I needed to make this system work with my Ember only frontend. This was a simple enough process but requires a bit of overriding the right points to make it work correctly. The reason why is because DV (really Devise in general) is built primarily for those who use Rails' rusty, old `views/` folder.

This post is setup as a helper for those who are building their top of the line frontend app on Rails and Devise and are looking to use that `:invitable` goodness. I'll provide a quick walkthrough on how to override the important parts to make it as painless as possible and you'll be onto your next user story in no time. It's going to focus on using Ember CLI and handlebars, but in case you're using React, Angular, or whatever else you're into it'll be easy enough to take the backend code and retrofit for your frontend of choice. Let's get to it!

## Setup

This post assumes you've got your Rails API, Ember CLI, Devise, and Ember Simple Auth App already setup to the point where you can use `rails console` to create a new `User` record and then sign them in from the frontend. If not, then I suggest you follow this [Ember + Devise tutorial](https://romulomachado.github.io/2015/09/28/using-ember-simple-auth-with-devise.html) and check back here once you're able to log in. It'll also assume you've done the [scambra/devise_invitable](https://github.com/scambra/devise_invitable) install, so be sure to follow the steps in that README.

Lastly on the setup front, if you're not setup to send emails locally from your Rails app then I suggest checking out [Letter Opener](https://github.com/ryanb/letter_opener). A handy little Gem originally created by Ryan Bates, it'll open any emails Rails sends locally in your web browser so you can test things out without any hassle. Very nice for development.

## The Backend

We'll start off by overriding the DV bits that need to be updated to work in a API Only environment. This main bit of code here is our override of DV's [`Devise::InvitationsController`](https://github.com/scambra/devise_invitable/blob/master/app/controllers/devise/invitations_controller.rb) (read if you're interested in what is going on under the hood). We're going to put our own `UsersInvitationsController` in it's place, so first let's generate that via `rails g controller users_invitations` and we'll end up with something like this:

```ruby
# backend/app/controllers/users_invitations_controller.rb
class UsersInvitationsController < ApplicationController
end
```

Let's update that real quick...

```ruby
# backend/app/controllers/users_invitations_controller.rb
class UsersInvitationsController < Devise::InvitationsController

  def edit
    sign_out send("current_#{resource_name}") if send("#{resource_name}_signed_in?")
    set_minimum_password_length
    resource.invitation_token = params[:invitation_token]
    redirect_to "http://localhost:8080/users/invitation/accept?invitation_token=#{params[:invitation_token]}"
  end

  def update
    super do |resource|
      if resource.errors.empty?
        render json: { status: "Invitation Accepted!" }, status: 200 and return
      else
        render json: resource.errors, status: 401 and return
      end
    end
  end
end
```

Okay, let's break that down a bit:
1. We extend `Devise::InvitationsController` so we've got all of it's actions as our base.
2. Add some code to override the actions we need for our API-only setup:
  - `edit`: This is normally the Rails rendered HTML page the user goes to enter their passwords and confirm signup. But we don't want to serve HTML out of Rails, so we grab the `invitation_token` from the url that came in and then redirect them to our frontend. Here I'm using localhost, but you'll likely redirect to a `Rails.config` var which is set per environment.
  - `update`: This is what the password form submit's to and would normally render a new HTML page with a flash message like 'You're signed up!'. But we want an JSON API only endpoint here, so we let the method do it's own thing (by calling `super`) and then take over to render a 200 or a 401 JSON blob according to if there was any errors or not.

Now that we understand what we're doing there we need to go ahead and tell Devise to use that controller instead of the existing `Devise::InvitationsController`. This is a simple edit to our `routes.rb` file:

```ruby
# backend/config/routes.rb
Rails.application.routes.draw do
  devise_for :users, controllers: { invitations: 'users_invitations' }

  # ...

  root to: 'application#root'
end
```

So we're now intercepting the important calls for our User invite process and turning them into JSON API endpoints. Cool!

Now let's move on to wiring up our frontend to stitch that invite flow together.

## The Frontend

Let's start off with adding a new Ember route to host our "Accept Invitation" experience. Let's generate that real quick via `ember g route accept-invitation` and then go ahead and give it a nicer URL (which also matches up to what we redirect to above in `UsersInvitationsController#edit`):

```javascript
// frontend/app/router.js
import Ember from 'ember';
import config from './config/environment';

const Router = Ember.Router.extend({
  location: config.locationType,
  rootURL: config.rootURL
});

Router.map(function() {
  // ...
  this.route('accept-invitation', { path: '/users/invitation/accept' });
});

export default Router;
```

Awesome, we should now be able to invite a user via `User.invite!(email: 'rick@ricknmortyforever1000years.com')`, get an email via `letter_opener`, click our sign up link in our email, and then have a blank ember route show up with no errors in the console. Pretty awesome!

Next we need to finish the invite loop by giving our user's a password form and then POST some JSON back to our `UsersInvitationsController#update` endpoint. Here is the code that gets us there:

```javascript
// frontend/app/routes/accept-invitation.js
import Ember from 'ember';
import config from '../config/environment';

const { inject: { service }, isEmpty } = Ember;

export default Ember.Route.extend({
  session: service('session'),
  beforeModel() {
    if (this.get('session.isAuthenticated')) {
      this.get('session').invalidate();
    }
  },

  model() {
    return Ember.Object.create({ password: null, password_confirmation: null, invitation_token: null });
  },

  afterModel(model, transition) {
    let invitationToken = transition.queryParams.invitation_token;
    if (isEmpty(invitationToken)) {
      this.transitionTo('dashboard');
    } else {
      model.set('invitation_token', invitationToken);
    }
  },

  actions: {
    submit(model) {
      let body = { user: model.getProperties('invitation_token', 'password', 'password_confirmation') };
      Ember.$.ajax({
        url: `${config.APP.API_HOST}/users/invitation`,
        type: 'PUT',
        data: body,
      }).then(() => {
        this.transitionTo('login');
      });
    }
  }
});
```

```handlebars
{{!-- app/templates/accept-invitation.hbs --}}
{{#bs-form model=model onSubmit=(route-action "submit" model) as |f|}}
  {{f.element controlType="password" label="Password" placeholder="Password" property="password" required=true}}
  {{f.element controlType="password" label="Password Confirmation" placeholder="Password Confirmation" property="password_confirmation" required=true}}
  {{bs-button defaultText="Submit" type="primary" buttonType="submit"}}
{{/bs-form}}

```

I'll break down the above to make sure we're clear on everything going on:
- We've got a `beforeModel` hook to sign the user out if they somehow have an existing session.
- We've got our `model` hook which returns an new `Ember.Object` with the properties we care about stubbed.
- We've got an `afterModel` hook to pull our `invitation_token` out of the URL and set it on our model object for later. This `invitation_token` is originally from the email and we included it in the redirect to our Ember App.
- Finally, our `submit` action. Here we pull the attributes from our model that we care about, build the specific JSON structure that the underlying `Devise::InvitationsController` is expecting (all properties under the `user` key is the important bit here), and then we send it off to our `#update` Rails endpoint that we overrode earlier. After this is done, we can transition our user over to login as they've have successfully completed the invite flow. Success!

## Expanding our invite Form

One last thing I'll add before I wrap things up. If you're looking to expand your invite form to accept other info from your user then you'll need to do a bit more work. Devise has its own way of doing controller param sanitization and you'll need to override that if you'd like to have your request get through with more than just the token and passwords. Here is the bit of code that you'll want to put in place:

```ruby
# backend/app/controllers/users_invitations_controller.rb
class UsersInvitationsController < Devise::InvitationsController
  before_action :configure_permitted_parameters

  # ...

  private

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:accept_invitation, keys: [:first_name, :last_name])
  end
end
```

## Wrapping up

Okay, so we've done some overriding at the Rails layer, created a simple route and form to accept our User's password info, and then we took that info and got it back to our server. We're using the great, solid functionality of `:invitable` but without having to serve HTML from our Rails application, which is a good win. If we need to expand our invite form in the future, we've got a way to do that as well.

That about wraps it up. If you've got questions or comments, please don't hesitate to ping me.
