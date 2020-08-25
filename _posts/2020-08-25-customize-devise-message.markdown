---
layout: post
title:  "Customize Devise's flash messages"
date:   2020-08-24 21:31:36 +0200
categories: jekyll update
---

**PROBLEM:**
I don't like the "Welcome, you have signed up successfully" flash message that **Devise** displays when a user signs up. I want something more personalized, I want a warm welcome. Easy! In a blink, it will be done!  Well, not that fast, actually.

### What do I need to display a personalized message?

To add a bit of warmness in this boring process of signing up, I want to address to the new user by her/his **username**.
So first I need to **modify my user model** by adding a username attribute.
```
$ rails generate migration AddUsernameToUsers username:string
```
Then to save the change of my DB schema:
```
$ rails db:migrate
```
Now I have to give my user the possibility to set a nice username. As far as I can see, the Devise's default views have only 2 required fields for signing up: email and password. Ok, so I need to **access the registrations views** to modify it I guess.

### Where are the Devise's registrations Views?

In my app folder, Devise's views are nowhere to be found. Obviously they are hidden, but it must be a way to see them!
Here comes the great documentation written by the authors of the gem[^1]. Devise provides a simple **generator** to copy all the views associated with authentification in a  dedicated folder.
```
$ rails generate devise:views
```
Now I have a Devise folder, and inside I can find a registration folder with the **new.html.erb** file I deserve.
I just need to add the username field on the form:
``` ruby
 <div class="form-inputs">
        <%= f.input :username,
                    required: true,
                    autofocus: true,
                    input_html: { autocomplete: "username" }%>
        <%= f.input :email,
                    required: true,
                    autofocus: true,
                    input_html: { autocomplete: "email" }%>
        <%= f.input :password,
                    required: true,
                    hint: ("#{@minimum_password_length} characters minimum" if @minimum_password_length),
                    input_html: { autocomplete: "new-password" } %>
        <%= f.input :password_confirmation,
                    required: true,
                    input_html: { autocomplete: "new-password" } %>
      </div>
 ```

### Where is the controller responsible for registration actions?

 When a user provides a correct email, password and username, a new user is registered,  Under the hood, the Registrations Controller is hit and specifically its create method that holds the logic for registration.  If the process is successful, the controller will ask the associated view to display a welcome message.
I need to access this controller to modify the create method in the way I want. Right, but by default, Devise's controllers are hidden as well. Not a problem, there is also a simple **generator to create customizable controllers**:
```
$ rails generate devise:controllers users
```
I specify the users as the scope, in order to keep the devise controllers separated from the others. The controllers associated with authentification will be in a users folder, inside the controllers folder of my app.

### How do I modify the Registrations Controller?

Now I can see my Registrations Controller, but not in the way I had imagined.
```
  # POST /resource
  # def create
  #   super
  # end
```
I know The 'super' keyword[^2] has something to do with parent/child classes. I could keep it and add a new behavior, but as I want to modify and not add, I will simply replace it by a custom code.
Well, I don't want to customize all of it, so I would prefer to keep the original code and modify what I need. A quick look to the **Devise's source of the Registrations Controller code**[^3] and here it goes:
``` ruby
# POST /resource

def create
  build_resource(sign_up_params)
  resource.save

  yield resource if block_given?
  if resource.persisted?
    if resource.active_for_authentication?
    set_flash_message! :notice, :signed_up
    sign_up(resource_name, resource)
    respond_with resource, location: after_sign_up_path_for(resource)
    else
    set_flash_message! :notice, :"signed_up_but_#{resource.inactive_message}"
    expire_data_after_sign_in!
    respond_with resource, location: after_inactive_sign_up_path_for(resource)
    end
  else
  clean_up_passwords resource
  set_minimum_password_length
  respond_with resource
  end
end
````
I have 2 possibilities here, depends if I plan to **internationalize** my app in the future.
If not, I could simply replace the
```
set_flash_message!  :notice,  :signed_up
```
by
```
flash[:notice] = "Hello there, #{resource.username}! Glad to welcome you here!"
```
But I do want to use I18n gem soon, so I prefer to pass the username variable to the set_flash_message method and then modify the **devise.en.yml** file that holds all the flash messages translations.
````
set_flash_message! :notice, :signed_up, :username => resource.username
````

### How do I customize the flash message?

Now that the set_flash_message method can access the username variable, I just need to open the **devise.en.yml** file, find the registration sign_up message and modify it:
```yml
en:
  devise:
    registrations:
      sign_up: "Hello there, %{username}! Glad to welcome you here!."
````

### Debugging...

Everything seems fine, I restarted the server, and now I am ready to receive my nice welcome message.
A quick test and well, how disappointing! I have the new flash message, yes, but I don't see my username appear. What can be wrong I ask? It was supposed to be super fast, and I already spent too much time understanding the whole thing.
Checking the rails console I realize that the last registered user has a **username set to NIL**. I entered a beautiful one though,  so Devise just refused to save it in my DB. Must be a security problem I guess, for sure Devise's methods have strong parameters. Just need to find where to add the username parameter to the list.

#### Devise strong parameters

Once again Devise's documentation is my friend. I discover that I don't need to modify  Devise's source code, but I have to **configure the permitted parameters** in the **Application Controller** in 2 steps.
- Adding a before action
```` ruby
class ApplicationController < ActionController::Base
  before_action :configure_permitted_parameters, if: :devise_controller?
````
- Adding a configure_permitted_parameters method
```ruby
  protected

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, keys: [:username])
  end
  ````

And now, the magic happens ;)

[^1]: All references for devise are from their great documentation. For example, to configure the views, see:[https://github.com/heartcombo/devise#configuring-views](https://github.com/heartcombo/devise#configuring-views)

[^2]: For a quick overview about the 'super' keyword in ruby, see: [https://medium.com/rubycademy/the-super-keyword-a75b67f46f05](https://medium.com/rubycademy/the-super-keyword-a75b67f46f05)

[^3]: Devise's source code is open source and available in the heartcombo github repository. For example, the Registration Controller's source code is here:[https://github.com/heartcombo/devise/blob/master/app/controllers/devise/registrations_controller.rb](https://github.com/heartcombo/devise/blob/master/app/controllers/devise/registrations_controller.rb)



<!-- Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/ -->
