## Google Sign in with Rails 7 and Omniauth 

First create a new rails app:

```shell
$ rails new google-omniauth-devise --css tailwind
```

Add the devise and omniauth gems to your Gemfile:

```ruby
gem "devise", "~> 4.8.1"
gem "omniauth"
gem "omniauth-google-oauth2"
gem "omniauth-rails_csrf_protection"
```

Install devise and omniauth gems with bundle and run the rails generator:

```
$ bundle install
$ rails generate devise:install
```

Create our user model for devise:

```
$ rails generate devise User
```

We'll need to add 2 fields to the migration to create our User table, open the migration file in the db folder and add the following 2 lines to the migration:

```
t.string :provider
t.string :uid
```

These fields will be used by omniauth when our user is signing in with Google oAuth

Run the migration to create our User model:

```
$ rails db:migrate
```

Let's create a home controller for our root path of our app:

```
$ rails g controller home index
```

We can edit the index view in app/views/home/index.html.erb and just have a welcome message placeholder:

```
<h2>Welcome</h2>
```

Next we need to login to the [Google Cloud console][google-cloud-console] and create a new project for our app. When in the new project, go to 'APIs and Services' and then 'Credentials'. Then create a new credential with type 'OAuth client ID'. Copy the the client id and client secret that are generated for this new credential.

Open the config/initializers/devise.rb file and add the following lines to the end of the config block:

```
config.omniauth :google_oauth2, "YOUR-NEW-CLIENT-ID", "YOUR-NEW-CLIENT-SECRET"
config.omniauth_path_prefix = "/users/auth"
```

Use the new client id and secret you create on the Google cloud console in the above block.

Edit the User model and add the omniauth options to the devise module inlude:

```
devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :omniauthable, omniauth_providers: [:google_oauth2]
```

Create a new controller in your project at app/controllers/users/omniauth_callbacks_controller.rb:

```
class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
    # See https://github.com/omniauth/omniauth/wiki/FAQ#rails-session-is-clobbered-after-callback-on-developer-strategy
    skip_before_action :verify_authenticity_token, only: :google_oauth2
  
    def google_oauth2
      # You need to implement the method below in your model (e.g. app/models/user.rb)
      @user = User.from_omniauth(request.env["omniauth.auth"])
  
      if @user.persisted?
        sign_in_and_redirect @user, event: :authentication # this will throw if @user is not activated
        set_flash_message(:notice, :success, kind: "Google") if is_navigational_format?
      else
        session["devise.google_data"] = request.env["omniauth.auth"].except(:extra) # Removing extra as it can overflow some session stores
        redirect_to new_user_registration_url
      end
    end
  
    def failure
      redirect_to root_path
    end
end
```

The google_oauth2 method will be called when we receive the callback from Google with the users data in the "omniauth.auth" hash.

Then we need to implement the from_omniauth method on our user model:


```
 def self.from_omniauth(access_token)
      data = access_token.info
      user = User.where(email: data['email']).first
    
      # Uncomment the section below if you want users to be created if they don't exist
      unless user
          user = User.create( 
              #uncomment below if you have a name field on your user model
              #name: data['name'],
              email: data['email'],
              password: Devise.friendly_token[0,20],
              provider: 'google_oauth2'
          )
      end
    
      user
  end
```

This implementation creates a user if they don't already exist in our database.

Finally, in our config/routes.rb we need to reference our omniauth callback controller in the devise_for config:

```
devise_for :users, path: 'u', controllers: {omniauth_callbacks: 'users/omniauth_callbacks'}
```

Devise will automatically create a button for signing in with Google in the links on our sign in page at http://localhost:3000/u/sign_in

If you want to customize your sign in and registration forms, you can copy them from devise into your app/views/devise folder:


```
$ rails g devise:views
```

[google-cloud-console]: https://console.cloud.google.com/
