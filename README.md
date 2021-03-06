# [handshakejs](https://handshakejs.herokuapp.com) API Documentation

![](https://rawgithub.com/handshakejs/handshakejs-script/master/handshakejs.svg)

**API platform for authenticating users without requiring a password.**

## Installation

### Heroku

```bash
git clone https://github.com/sendgrid/handshakejs-api.git
cd handshakejs-api
heroku create handshakejs
heroku addons:add sendgrid
heroku addons:add redistogo
git push heroku master
heroku config:set FROM=login@yourapp.com
```

Next, create your first app.

```bash
curl -X POST https://handshakejs.herokuapp.com/api/v0/apps/create.json \
-d "email=you@email.com" \
-d "app_name=your_app_name"
```

Nice, that's all it takes to get your authentication system running. Now let's plug that into our app using the embeddable JavaScript.

Place a script tag wherever you want the login form displayed.  

```html
<script src='/path/to/handshake.js' 
        data-app_name="your_app_name" 
        data-root_url="https://handshakejs.herokuapp.com"></script>
```

Get the latest [handshake.js here](https://github.com/sendgrid/handshakejs-script/blob/master/build/handshake.js). Replace the `data-app_name` with your own.

Next, bind to the handshake:login_confirm event to get the successful login data. This is where you would make an internal request to your application to set the session for the user.

```html
<script>
  handshake.script.addEventListener('handshake:login_confirm', function(e) {
    console.log(e.data);
    $.post("/login/success", {email: e.data.identity.email, hash: e.data.identity.hash}, function(data) {
      window.location.href = "/dashboard";
    });    
  }, false); 
</script>
```

Then you'd setup a route in your app at /login/success to do something like this (setting the session). Here's an example in ruby and there is also a [full example ruby app](https://github.com/handshakejs/handshakejs-example-ruby).

```ruby
  post "/login/success" do
    salt    = "the_secret_salt_when_you_created_an_app_that_only_you_should_know"
    pbkdf2  = PBKDF2.new(:password=>params[:email], :salt=>salt, :iterations=>1000, :key_length => 16, :hash_function => "sha1")

    session[:user] = params[:email] if pbkdf2.hex_string == params[:hash]
    redirect "/dashboard"
  end
```

### Click to cloud (beta)

You can optionally install using `click-to-cloud`. Click to cloud is a binary I'm building to make it easier to deploy
small application to cloud Paas like Heroku. I personally, use this approach, but your mileage may vary. 

First, [install click-to-cloud](https://github.com/scottmotte/click-to-cloud#installation) on your machine.

Second, run the following command.

```bash
click-to-cloud --repo https://github.com/sendgrid/handshakejs-api.git
```

That's it. That will install your application to Heroku.

## API Overview

The [handshakejs.herokuapp.com](https://handshakejs.herokuapp.com) API is based around REST. It uses standard HTTP authentication. [JSON](https://www.json.org/) is returned in all responses from the API, including errors.

I've tried to make it as easy to use as possible, but if you have any feedback please [let me know](mailto:scott@scottmotte.com).

* [Summary](#summary)
* [Apps](#apps)
* [Login](#login)

## Summary

### API Endpoint

* https://handshakejs.herokuapp.com/api/v0

## Apps

To start using the handshake API, you must first create an app.

### POST /apps/create

Pass an email and app_name to create your app at handshakejs.herokuapp.com.

#### Definition

```bash
POST https://handshakejs.herokuapp.com/api/v0/apps/create.json
```

#### Required Parameters

* email
* app_name

#### Optional Parameters

* salt

#### Example Request

```bash
curl -X POST https://handshakejs.herokuapp.com/api/v0/apps/create.json \
-d "email=test@example.com" \
-d "app_name=myapp"
```

#### Example Response
```javascript
{
  success: true,
  app: {
    email: "test@example.com",
    app_name: "myapp",
    salt: "the_default_generated_salt_that_you_should_keep_secret"
  }
}
```

## Logins

### POST /login/request

Request a login.

#### Definition

```bash
POST https://handshakejs.herokuapp.com/api/v0/login/request.json
```

#### Required Parameters

* email
* app_name

#### Example Request

```bash
curl -X POST https://handshakejs.herokuapp.com/api/v0/login/request.json \ 
-d "email=test@example.com" \
-d "app_name=your_app_name"
```

#### Example Response
```javascript
{
  success: true,
  identity: {
    email: "test@example.com",
    app_name: "your_app_name",
    authcode_expired_at: "1382833591309"
  }
}
```

### POST /login/confirm

Confirm a login. Email and authcode must match to get a success response back. 

#### Definition

```bash
POST https://handshakejs.herokuapp.com/api/v0/login/confirm.json
```

#### Required Parameters

* email
* authcode
* app_name

#### Example Request

```bash
curl -X POST https://handshakejs.herokuapp.com/api/v0/login/confirm.json \
-d "email=test@example.com" \
-d "authcode=7389" \ 
-d "app_name=your_app_name"
```

#### Example Response
```javascript
{
  success: true,
  identity: {
    email: "test@example.com",
    app_name: "your_app_name",
    authcode: "7389"
  }
}
```

## Database Schema with Redis

apps - collection of keys with all the app_names in there. SADD

apps/myappname - hash with all the data in there. HSET or HMSET

apps/theappname/identities - collection of keys with all the identities' emails in there. SADD

apps/theappname/identities/emailaddress HSET or HMSET

