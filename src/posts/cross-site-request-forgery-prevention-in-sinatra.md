+++
title = "Cross-Site Request Forgery Prevention in Sinatra"
date = 2014-08-21
tags = ["CSRF", "Sinatra", "Security"]
aliases = ["/2014/08/cross-site-request-forgery-prevention-in-sinatra.html"]
+++

Unlike Rails, [Sinatra](https://github.com/sinatra/sinatra) has no native
cross-site request forgery prevention so I thought I'd take a minute to outline
the implementation that I use to prevent these kinds of attacks. It's pretty
naive and simple, but that's the kind of implementation I like.

If you're unaware, [cross-site request forgery](csrf) (CSRF) is a _very_ common
vulnerability that exploits an existing session in a user's browser and the
inherent trust that establishes with the server to carry out malicious acts.
Luckily, it's fairly easy to prevent once you know about it.

## A Pattern for Implementation

The implementation pattern I picked is called the
"[Synchronizer Token Pattern](stp)". Basically, we generate a token server-side
that the client should send along with every request, thus (hopefully) ensuring
trust between the client and server. The token needs to be
[really, truly random](random) to avoid predictability and should be sent to the
client in (at least) the HTML of the document. I send the token along as an
HTTP-Only cookie as well to add another layer of extra-double-sureness to the
prevention mechanism, but you don't have to if you don't want to.

## Implementation

Code-wise, this is all in Sinatra and depends on having some session setup.

### The Layout

```html
<meta name="csrf-token" content="<%= session[:csrf] %>" />
```

### The JavaScript

Here, we hook into the submit event of any form and add the CSRF token to
anything that isn't a GET request:

```coffeescript
$(document).on 'submit', 'form', ->
  form   = $(this)
  method = form.attr('method').toUpperCase()
  token  = $('meta[name=csrf-token]').attr('content')

  # add the CSRF token as a hidden input to the form
  if method? and method isnt 'GET'
    form.prepend $('<input>', name: '_csrf', type: 'hidden', value: token)
```

### The Sinatra App

```ruby
before do
  session[:csrf] ||= SecureRandom.hex(32)

  response.set_cookie 'authenticity_token', {
    value: session[:csrf],
    expires: Time.now + (60 * 60 * 24 * 180), # that's 180 days
    path: '/',
    httponly: true
    # secure: true # if you have HTTPS (and you should) then enable this

  # this is a Rack method, that basically asks
  #   if we're doing anything other than GET
  if !request.safe?
    # check that the session is the same as the form
    #   parameter AND the cookie value
    if session[:csrf] == params['_csrf'] && session[:csrf] == request.cookies['authenticity_token']
      # everything is good.
    else
      halt 403, 'CSRF failed'
    end
  end
end
```

[csrf]: https://en.wikipedia.org/wiki/Cross-site_request_forgery
[random]:
	https://en.wikipedia.org/wiki/Random_number_generation#.22True.22_random_numbers_vs._pseudo-random_numbers
[stp]:
	https://en.wikipedia.org/wiki/Cross-site_request_forgery#Synchronizer_Token_Pattern
