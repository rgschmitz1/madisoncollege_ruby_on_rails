footer:@johnsonch :: Chris Johnson :: Ruby on Rails Development :: Week 10
autoscale: true

#Ruby on Rails Development
##Week 10 

* https://codecaster.io room: johnsonch

---

###If you need to re-clone
* ```$ git clone git@bitbucket.org:johnsonch/wolfiereader.git```
* ```$ cd wolfiereader```
* ```$ bundle install --without production```

###If you have it already cloned
* cd into ```wolfiereader```
* ```$ git add . ```
* ```$ git commit -am 'commiting files from in class'```
* ```$ git checkout master```
* ```$ git fetch```
* ```$ git pull ```
* ```$ git checkout week10_start```
* ```$ rm -f db/*.sqlite3```
* ```$ bundle```
* ```$ rake db:migrate```

---

###Fixing the auto association


open app/controllers/feeds_controller.rb and we're going to adjust the create
action to use create and not save.  When we use save we're only saving the object
but when we use create Rails will take care of creating the association for us.

```ruby
  def create
    
    feed = @user.feeds.new(feed_params) # Making a new object so we can validate it before tyring to create

    respond_to do |format|
      if feed.valid?
        @user_feed = @user.feeds.create(feed_params) #Using create because it saves the association for us
        format.html { redirect_to user_feed_path(id: @user_feed.id), notice: 'Feed was successfully created.' }
        format.json { render :show, status: :created, location: @user_feed }
      else
        format.html { render :new }
        format.json { render json: @user_feed.errors, status: :unprocessable_entity }
      end
    end
  end
```


Now that our auto association works we can start implementing the login and out system.

```bash
$ rails generate controller Sessions new
```

With our sessions controller generated let's add all the routes we'll need

```ruby
  get    'login'   => 'sessions#new'
  post   'login'   => 'sessions#create'
  delete 'logout'  => 'sessions#destroy'
```

Next up let's get a form in place for users to fill out their login information.
We'll work in app/views/sessions/new.html.erb

```ruby

<h1>Log in</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3 well">
    <%= form_for(:session, html: {class: "form-horizontal"}) do |f| %>
      <fieldset></fieldset>

      <div class="form-group">
        <%= f.label :email, class: 'col-lg-2 control-label' %>
        <div class="col-lg-10">
          <%= f.text_field :email, class: 'form-control' %>
        </div>
      </div>

      <div class="form-group">
        <%= f.label :password, class: 'col-lg-2 control-label' %>
        <div class="col-lg-10">
          <%= f.password_field :password, class: 'form-control' %>
        </div>
      </div>

      <div class="form-group">
        <div class="col-lg-12">
          <%= f.submit "Login", class: "btn btn-primary" %>
        </div>
      </div>

    <% end %>
    <p>New user? <%= link_to "Sign up now!", signup_path %></p>
  </div>
</div>

```

Next we'll need the build the controller action to handle our form post.  If we
were to try and fill it out now we would end up getting an error.  We can try
and see what that error looks like. It will help you with learning how to read
error messages.  Then we'll open up our sessions controller and add the following
method:

```ruby
  def create
    puts params.inspect
    render 'new'
  end
```

Let's try out the form to see it work, you'll see some output in your log like this:

```
Started POST "/login" for ::1 at 2016-03-30 22:47:06 -0500
Processing by SessionsController#create as HTML
  Parameters: {"utf8"=>"‚úì", "authenticity_token"=>"ll590vaPw1MwrLOg+sd3ZhI/qecz1ffYaMTlwq+MO2wh0Gv8DsmRn9SPbBQthdjPZnCKDgwaKHe67GhHbfl/uA==", "session"=>{"email"=>"marge@simpsons.com", "password"=>"[FILTERED]"}, "commit"=>"Login"}
{"utf8"=>"‚úì", "authenticity_token"=>"ll590vaPw1MwrLOg+sd3ZhI/qecz1ffYaMTlwq+MO2wh0Gv8DsmRn9SPbBQthdjPZnCKDgwaKHe67GhHbfl/uA==", "session"=>{"email"=>"marge@simpsons.com", "password"=>"P@ssw0rd!"}, "commit"=>"Login", "controller"=>"sessions", "action"=>"create"}
  Rendered sessions/new.html.erb within layouts/application (1.4ms)
  Rendered layouts/_navigation.html.erb (0.3ms)
Completed 200 OK in 52ms (Views: 51.0ms | ActiveRecord: 0.0ms)
```

Now that we know for sure what our form is sending we can use that information
to log our user in.

```ruby
  def create
    user = User.find_by(email: params[:session][:email])
    if user && user.authenticate(params[:session][:password])
    else
      flash.now[:danger] = 'Invalid email/password combination'
      render 'new'
    end
  end
```

Now we'll add some methods to our ```SessionHelper``` to make logging in and out
easier.  In order for it to be useful to our controller we'll need to make it 
available by including it.  We'll do that in our application controller.

```ruby
include SessionsHelper
```

In our sessions helper we'll start by adding a ```log_in``` method

```ruby
  def log_in(user)
    session[:user_id] = user.id
  end
```

Then we can use it in our sessions controller, and then after the user is logged
in we can redirect them to their page to view all their feeds.

```ruby
  def create
    user = User.find_by(email: params[:session][:email])
    if user && user.authenticate(params[:session][:password])
      log_in(user)
      redirect_to(user)
    else
      flash.now[:danger] = 'Invalid email/password combination'
      render 'new'
    end
  end
```

Next we can make some future operations easier by implementing the concept of 
the current user.

In our SessionsHelper we'll add the following:

```ruby
def current_user
  if @current_user.nil?
    @current_user = User.find_by(id: session[:user_id])
  else
    @current_user
  end
end
```

^ Instance variables save us from hitting the database for each time we load the
page. If we didn't use them then every call to current user would hit the database
but this way we set an instance variable for the remainder of the request causing
only one DB hit per page load.


Let's add some links that change when you are logged in!  But first we'll need
a helper to verify if someone is logged in or not:

```ruby
  def logged_in?
    !current_user.nil?
  end
```

Here we have a full updated version of the navigation list. 

```ruby
<nav class="navbar navbar-default navbar-static-top" role="navigation">
  <div class="navbar-header">

    <button type="button" class="navbar-toggle" data-toggle="collapse" data-target="#bs-example-navbar-collapse-1">
      <span class="sr-only">Toggle navigation</span><span class="icon-bar"></span><span class="icon-bar"></span><span class="icon-bar"></span>
    </button> <a class="navbar-brand" href="#">Brand</a>
  </div>

  <div class="collapse navbar-collapse" id="bs-example-navbar-collapse-1">
    <ul class="nav navbar-nav">
      <li class="active">
        <%= link_to 'Home', root_path %>
      </li>
      <li>
        <%= link_to 'About', about_path %>
      </li>
      <li>
        <%= link_to 'Opensource at WolfieReader', opensource_path %>
      </li>
      <% if logged_in? %>
        <li>
          <%= link_to 'Dashboard', current_user %>
        </li>
        <li>
          <%= link_to 'Logout', logout_path, method: "delete" %>
        </li>
      <% else %>
        <li>
          <%= link_to 'Login', login_path %>
        </li>
        <li>
          <%= link_to 'Register', new_user_path %>
        </li>
      <% end %>
    </ul>
  </div>

</nav>
```


Then we can log people in upon sign up by adding the ```log_in``` method to the
create user action

```ruby
log_in(@user)
```

Then we can handle logout by adding a logout method to our sessions helper

```ruby
  def log_out
    session.delete(:user_id)
    @current_user = nil
  end
```

And then a destroy method to our sessions controller

```ruby
 def destroy
    log_out
    redirect_to root_url
  end
```
