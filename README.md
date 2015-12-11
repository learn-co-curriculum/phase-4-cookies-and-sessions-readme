# Cookies and sessions

## Overview

Cookies are a way for an HTTP server to ask the user's browser to store a little bit of data for it, and then get that data back from the browser later.

They are fundamental to the operation of nearly every contemporary website.

Primarily, cookies are used for log in. They provide a way for us to verify who a user is once, and then remember it for their entire session. Without cookies, you would have to provide your username and password on every single request to the server.

Cookies may also be used to store other information about a user, such as what's in their shopping cart, or what ads you've shown them during their visit.

In this document, we'll cover what cookies are, how they fit into the HTTP response flow, and how you can access them from Rails.

## Objectives
  1. Describe a cookie.
  2. Explain the data flow problem cookies solve.
  3. Find all the cookies on a page.
  4. Explain what a session is, as far as Rails is concerned.
  5. Describe where session data is stored.
  6. Write data to the Rails session and access it later
     in the app.

## Shopping

Let's say we want to write an e-commerce site. Users can come to the site, add some items to their shopping cart, and then pay for all of them at the end. Simple.

Where does the user's shopping cart live?

We could create a `Cart` model, which `has_and_belongs_to_many` `Item`s. But that leaves a fundamental problem: how do we know what cart to load when we get a request?

Remember what's included in an HTTP request:
  * an [HTTP verb][rfc_http_methods], like `GET`, `PUT`, or `POST`
  * a path
  * various headers

HTTP servers are typically stateless. Your Rails server has access to a database, but the request needs to have information about what database rows to look up.

For example, `GET` requests usually encode this information in the path. When you write a route matching `/items/:item_id`, you are telling Rails to pull the value `id` from the request path and save it in `params[:id]`. In your `items_controller`, you'll probably have a method that looks something like:

    def show
      @item = Item.find!(params[:item_id])
    end

Which loads the row for that item from the database and returns it as an ActiveRecord model object, which your `show.html.erb` then renders.

If we want to be able to retrieve the current cart, we need to its id to be somewhere in the HTTP request. Specifically, it must be in the path or the headers.

It would be possible, though quite convoluted, to store this information in the path. This would have strange effects: since the path is shown in the browser's URL bar, a user who copies a URL and sends it to a friend ("check out this neat skirt!") would also be copying their shopping cart ID. Upon loading the page, the friend would
see what's in the user's cart. That is probably not what we want.

Cookies allow us to store this information in the only other place available to us: HTTP headers.

## What's a cookie, anyway?

Let's see what [the spec][rfc_cookies] has to say:

      This section outlines a way for an origin server to send state
      information to a user agent and for the user agent to return the
      state information to the origin server.

      To store state, the origin server includes a Set-Cookie header in an
      HTTP response.  In subsequent requests, the user agent returns a
      Cookie request header to the origin server.  The Cookie header
      contains cookies the user agent received in previous Set-Cookie
      headers.  The origin server is free to ignore the Cookie header or
      use its contents for an application-defined purpose.

Their example is quite simple:

      == Server -> User Agent ==
      Set-Cookie: SID=31d4d96e407aad42

      == User Agent -> Server ==
      Cookie: SID=31d4d96e407aad42

In this example, the server responds to a request with the `Set-Cookie` header. This header sets the value of the `SID` cookie to `31d4d96e407aad42`.

When the user visits another page on the same server, the browser sends the cookie back to the server, including the `Cookie: SID=31d4d96e407aad42` header in its request.

Cookies are stored in the browser. The browser doesn't care about what's in the cookies you set. It just stores the data and sends it along on future requests to your server.

## Using cookies

So how would we use a cookie to store a reference to the user's shopping cart? Let's say that we create a cart the first time a user adds something to their cart. Then, in the response, we might include the header,

      == Server -> User Agent ==
      Set-Cookie: cart_id=273

Only with the `cart.id` of the cart we just saved.

When the user comes back to our site, their browser will include the cookie in their
reply:

      == User Agent -> Server ==
      Cookie: cart_id=273

We can look at this HTTP header, get the `cart_id` from it, and look it up using the `ActiveRecord` find method we know and love.

## Security concerns

Cookies are stored as plain text in a user's browser. Therefore, the user can see what's in them, and they can set them to anything they want.

If you open the developer console in your browser, you can see the cookies set by the current site. In Chrome's console, you can find this under `Resources > Cookies`. You can delete any cookie you like. For example, if you delete your `user_session` cookie on `github.com`, you will find that you've been logged out.

You can also edit cookies, for example with [this extension|edit_this_cookie].

This presents a problem for us. If users can edit their `cart_id` cookie, then they can see other users' shopping carts.

## Rails to the rescue

Fortunately, Rails has a solution to this. When you set cookies in Rails, you usually don't manipulate the HTTP headers directly. Instead, you use the `session` method. The `session` method is available anywhere in the Rails response cycle, and it behaves like a hash:

    # set cart_id
    session[:cart_id] = @cart.id

    # load the cart referenced in the session
    @cart = session[:cart_id]

You can store any simple Ruby object in the session. In fact, we don't need a `Cart` model at all—we can just store a list of items in the session!

Rails manages all session data in a single cookie, named `_YOUR_RAILS_APP_NAME_session`. It *serializes* your all the key/value pairs you set with `session`, turning them into a big string. Whenever you set a `key` with the `session` method, Rails updates the value of its session cookie to this big string.

When you set cookies this way, Rails signs them to prevent users from tampering with them. Your Rails server has a key, configured in `config/secrets.yml`.

    development:
      secret_key_base: kaleisgreat  # probably not the most secure key ever

Somewhere else, Rails has a method, let's call it `sign`, which takes a `message` and a `key` and returns a `signature`, which is just a string:

    # sign(message: string, key: string) -> signature: string
    def sign(message, key):
      # cryptographic magic here
      return signature
    end

It's guaranteed that given the same message and key, sign will produce output. Also, without the key, it is practically impossible to know what `sign` would return for a given message. That is, signatures can't be forged.

Rails creates a signature for every cookie it sets, and appends the signature to the cookie.

When it receives a cookie, Rails verifies that the signature matches the content (that is, that `sign(cookie_body, secret_key_base) == cookie_signature`).

This prevents cookie tampering. If a user tries to edit their cookie and change the `cart_id`, they signature won't match, and Rails will silently ignore the cookie and set a new one.

# Tying it together

In our `items_controller.rb`, we might have an `add_to_cart` method, which is called
when the user adds something to their cart. It might work something like this:

    # Routed from POST /items/:id/add_to_cart
    def add_to_cart
      # Get the item from the path
      @item = Item.find(params[:id])
      
      # Load the cart from the session, or create a new empty cart.
      cart = session[:cart] || []
      cart << @item.id

      # Save the cart in the session.
      session[:cart] = cart
    end

That's it! It's common to wrap this up in a helper method:

It's common to wrap up some of this functionality in a helper method:

    class ApplicationController < ActionController::Base
      def current_cart
        session[:cart] ||= []
      end
    end

So now our controller looks like this:

    # Routed from POST /items/:id/add_to_cart
    def add_to_cart
      # Get the item from the path
      @item = Item.find(params[:id])
      
      # Load the cart from the session, or create a new empty cart.
      current_cart << @item.id
    end

This way, we can use `current_cart` in our views and layouts too. For example, we may want to show the user how many items are in their cart as part of the layout.

## Conclusion

Cookies are foundational for the modern web.

You've probably heard a lot about cookies from non-technical sources. In fact, the EU recently passed a law requiring sites that use cookies to request user consent before doing so.

The effect of this law was not unlike the effect of California's [Proposition 65][prop_65], which states that buildings must have clear signage if they contain any one of a list of chemicals known to the state of California to cause cancer.
Nobody knows or wants to put in the considerable expense to figure out what's in their building, and signs are basically free, so literally every building in California now carries a prominent sign telling you it causes cancer.

Most sites use cookies, either to let their users log in, or to keep track of their
shopping carts, or record other ephemeral session data. Almost nobody thinks these are bad uses of cookies: nobody really believes that you should have to type in your username and password on every page, or that your shopping cart should clear if you reload the page.

But cookies just let you store data on in a user's browser, so by nature, they can be used for more controversial endeavors. 

For example, Google AdWords sets a cookie and uses that cookie to track what ads you've seen and which ones you've clicked on. The tracking information helps AdWords decide what ads to show you.

This is why, if you click on an ad, you may find that the ad follows you around the Internet. It turns out that this behavior is as effective as it is annoying: people are far more likely to buy things from ads that they've clicked on once.

This use of cookies worries people. And so now the EU has a law.

Cookies, like any technology, are a tool. In the rest of this unit, we're going to be using them to let users log in. Whether you later want to use them in such a way that the EU passes another law is up to you.

## Resources

  * [HTTP RFC Section 9 — Methods][rfc_http_methods]
  * [RFC 6265 — HTTP State Management Mechanism (the cookie spec)][rfc_cookies]
  * [Rails – Accessing the Session][rails_session]
  * [California Prop 65][prop_65]

[rfc_http_methods]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html "HTTP RFC 9 — Method Definitions"
[rfc_cookies]: http://tools.ietf.org/html/rfc6265 "HTTP State Management Mechanism"
[edit_this_cookie]: https://chrome.google.com/webstore/detail/editthiscookie/fngmhnnpilhplaeedifhccceomclgfbg?hl=en
[rails_session]: http://guides.rubyonrails.org/action_controller_overview.html#accessing-the-session
[prop_65]: http://oehha.ca.gov/prop65/background/p65plain.html
