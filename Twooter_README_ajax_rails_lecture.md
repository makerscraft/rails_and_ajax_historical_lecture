# Rails and AJAX

## Introduction

I've worked with Rails for a number of years and watched it remain
_uncharacteristically_ unopinionated with respect to characterizing the right
way of integrating JavaScript.  Lack of strong opinion in the framework creates
difficulty with regard to instruction.  Should students use `respond_to` and
`js.erb`, or should we use AJAX to render partials?  How can we help students
steer away from hodge-podges of various techniques that create Frankencode?
How to understand the role of CoffeeScript or Turbolinks?

In this document I want to take a look at how AJAX and Rails integration
evolved from a historical / discovery perspective.  This guide leads the reader
to build up an application that allows the reader to "re-discover" the
conversations and decisions Rails developers made while integrating the various
AJAX approaches into the framework.

As a warning, there are several deprecated approaches, conflicting approaches ,
and areas of "weak" opinion.  The reader should be astute and use these
discussions to think critically about the approach undertaken.  It's important
to know each of these approaches since it is likely, owing to the
"un-opinionatedness" of Rails on this matter, to encounter mixtures of these
techniques "in the wild."

## Twooter

Twooter is a basic application that demonstrates the basic principles of
interoperation between JavaScript with Rails.  We're going to let errors guide
us (error-driven development?) into understanding how we can work with Rails
_via_ JavaScript.

### Creating Twooter

1. `rails new -d postgresql -T twooter`
1. `cd twooter`
1. `git init; git add . ; git commit -m 'Initial Commit'`
1. `rails generate scaffold twoot body:string`
1. `git add db app/models/ app/controllers/ config/ app/views/twoots`
1. `git commit -vm 'Add Twoot model'`
1. `git clean -fd # remove garbage from stupid generator; don't use this!`
1. Correct `config/database.yml`
1. `rake db:create:all`
1. `rake db:migrate`
1. `rails s`

### Test Twooter with `curl`

As a starting point, let's try taking a look at a web page from `curl` so that
we can see what the browser is receiving.

1.  `rails s` in one window
1  In another window: `curl http://localhost:3000/twoots`

You should see the request in your sever log **and** you should see the HTML
representation of the page printed to the console where you issued the `curl`
command.

Now, let's get a more traditional view of a web page.  Visit
`http://localhost:3000/twoots` and use the web interface make a few `Twoot`s.
Now visit the same URL and you should see the members of `/twoots` which
corresponds to `TwootsController#index`.  Try revisiting the URL with `curl`
and see how the raw HTML view has changed.

Now it certainly makes sense that web browser users would want the HTML
document that, when parsed by a web browser, would create a beautifully
rendered web page. But what about API consumers?  Or people using `curl` or
other low-level browswers?  Wouldn't it be nice if these users could tell the
server that they wanted a different experience *but* the programmers could
decide how to render these two _different_ views in a single controller action?

### `respond_to` to the Rescue

In Rails 1.1.1, the method `[respond_to][111_respond_to]` was added to `
ActionController::MimeResponds::InstanceMethods`.  This method, which takes a
block, allows the programmer to express the following:

> "[I]f the client wants HTML in response to this action, just respond as we would
> have before, but if the client wants XML, return them the list of people in XML
> format."

Obviously, this interface could support an arbitrary collection of formats:
JSON, XML, JavaScript, HTML, and even custom formats.  It is a powerful way to
succinctly request, from the client, that an action return a representation
_other_ than HTML.

Let's take a look at how the client could express the intention of wanting
another format.  When we submit a web request we supply with it a
`Content-Type` header.  We can see this with `curl`.

    $ curl --head http://localhost:3000/twoots |sort

    ...snip...
    Connection: Keep-Alive
    Content-Length: 0
    Content-Type: text/html; charset=utf-8 <==== Watch this!
    Date: Tue, 24 Jun 2014 13:14:25 GMT
    HTTP/1.1 200 OK
    Server: WEBrick/1.3.1 (Ruby/2.1.2/2014-05-08)
    path=/; HttpOnly
    Set-Cookie: request_method=HEAD; path=/
    X-Content-Type-Options: nosniff
    X-Frame-Options: SAMEORIGIN
    ...snip...

By default, `curl` asks for a content type of HTML.  This makes sense as most
people want `curl` to represent what a browser would render.  But, it turns out
we can change the content type we request.

    $ curl -H "Accept: application/json" --head http://localhost:3000/twoots |sort

    Connection: Keep-Alive
    Content-Length: 0
    Content-Type: application/json; charset=utf-8 <=== See!
    Date: Tue, 24 Jun 2014 13:18:25 GMT
    Etag: "66e7191ae4878d4d00639aa8bb671871"
    HTTP/1.1 200 OK
    Server: WEBrick/1.3.1 (Ruby/2.1.2/2014-05-08)
    Set-Cookie: request_method=HEAD; path=/
    X-Content-Type-Options: nosniff
    X-Frame-Options: SAMEORIGIN
    X-Request-Id: b9a5c877-4b6f-44eb-b6a2-f62510a67a13
    X-Runtime: 0.005860
    X-Xss-Protection: 1; mode=block

Look at the `Content-Type`, it has changed.  Nice!

If we remove the `--head` flag we now see the actual JSON.

    $ curl -H "Accept: application/json" http://localhost:3000/twoots |sort

    [{"id":1,"body":"Help","url":"http://localhost:3000/twoots/1.json"},{"id":2,"body":"I'm
    twooting!","url":"http://localhost:3000/twoots/2.json"}]

Modern Rails version automatically support JSON, so let's try XML.

    $ curl -H "Accept: application/xml" http://localhost:3000/twoots |sort

We get this error:

    Started GET "/twoots" for 127.0.0.1 at 2014-06-24 06:23:07 -0700
    Processing by TwootsController#index as XML
    Completed 500 Internal Server Error in 1ms

    ActionView::MissingTemplate (Missing template twoots/index, application/index
    with {:locale=>[:en], :formats=>[:xml], :variants=>[], :handlers=>[:erb,
    :builder, :raw, :ruby, :jbuilder, :coffee]}. Searched in:
      * "/Users/sgharms/Desktop/twooter/app/views"

Something in the controller wants to do something helpful, and it knew I wanted
XML, but it couldn't find information about how to render these data.  I'll
make use of `respond_to` now.

Let's update `TwootsController#index`:

    def index
      @twoots = Twoot.all

      respond_to do |format|
        format.html
        format.xml { render xml: @twoots.to_xml }
      end
    end

In `respond_to`, last match within the block wins.  We try matching HTML, but
we fail, so we retry with XML where we succeed.  The block adjacent to the
`format.xml` line is added to the render queue.  When `#index` finishes, the
render action in the queue is executed, in this case rendering a text version
of all the `Twoot`s.

    $ curl -H "Accept: application/xml" http://localhost:3000/twoots

    <?xml version="1.0" encoding="UTF-8"?>
    <twoots type="array">
      <twoot>
        <id type="integer">1</id>
        <body>Help</body>
        <created-at type="dateTime">2014-06-24T13:09:10Z</created-at>
        <updated-at type="dateTime">2014-06-24T13:09:10Z</updated-at>
      </twoot>
      <twoot>
        <id type="integer">2</id>
        <body>I'm twooting!</body>
        <created-at type="dateTime">2014-06-24T13:09:18Z</created-at>
        <updated-at type="dateTime">2014-06-24T13:09:18Z</updated-at>
      </twoot>
    </twoots>

And thus Rails laid the stage for AJAX integration: it was now possible for
browsers and XML requesting objects (like an XMLHttpRequest object, the "X" in
AJAX) to ask the back-end for shared, uniform consistent data.

Rails developers of the era of the early aughts asked this question: "What if
an XMLHttpRequest object were to request JavaScript which it could then
execute?  If that were the case one could use server side templating
technologies (the ERB tags) to build JavaScript data on the server so that very
minimal / no JavaScript would have to be written on the client.  We don't like
JavaSript because we remember the late 90's and how horrible JavaScript was."

# In the year 2005 or so: "server-side" JavaScript

As the monologue above implies, at the dawn of the new millenium, Rails
developers viewed JavaScript with a lot of suspicon.  JavaScript was only
deemed worthy of doing funny effects (like window rollup of words and things
like that).  The preferred library for such eye-candy was
[scriptaculous][]Consequently the mental "picture" of a JavaScript behavior was
to do some client-side eye candy (window shade a `<div>`, etc.).

To that end, Rails developers thought it would be great to avoid writing
client-side JavaScript by embedding the simple JavaScript behavior inside of
formatted responders that were triggered by calling a Rails controller action.

## Howto:

0.  Base case.  Start with a simple application like Twooter.  In Twooter an
anonymous user "twoots" when she fills in a form (`twoots#new`) and then
submits the `POST` request (`twoots#create`).  If you observe the server log
while creating a Twoot, you'll see the following:

```text
    Started POST "/twoots" for 127.0.0.1 at 2014-06-21 14:38:02 -0400
    Processing by TwootsController#create as HTML
      Parameters: {"utf8"=>"✓",
    "authenticity_token"=>"zXWgClq1GTM/rrB+p/othEJsGcdF61r+ERmm+FH7ojo=",
    "twoot"=>{"body"=>"This is a new twoot"}, "commit"=>"Create Twoot"}
       (0.1ms)  BEGIN
      SQL (6.5ms)  INSERT INTO "twoots" ("body", "created_at", "updated_at") VALUES
    ($1, $2, $3) RETURNING "id"  [["body", "This is a new twoot"], ["created_at",
    Sat, 21 Jun 2014 18:38:02 UTC +00:00], ["updated_at", Sat, 21 Jun 2014 18:38:02
    UTC +00:00]]
       (14.0ms)  COMMIT
    Redirected to http://localhost:3000/twoots/154
    Completed 302 Found in 25.7ms (ActiveRecord: 20.8ms)
```

it is of note that the `Processing` line notices that we asked for this action
to happen from the context of HTML (`...as HTML`).  This implies that we might be
able to change the context format to something else.

1. Make your `POST` request a response in JavaScript (similar to how we
adjusted the `Content-Type` with `curl` above).  There are a number of ways to
accomplish this, but one of the easiest is to pass a `format` in `form_for`.
We'll look in the code for the form on the `app/views/twoots/new.html.erb`
action:

```ruby
  <%= form_for(@twoot, :format => :js) do |f| %>
```

Here I added a 'js' format prompt.  This tells Rails that I expect a response
in JavaScript.

2.  The request now looks like:

```text
    Started POST "/twoots.js" for 127.0.0.1 at 2014-06-21 14:42:20 -0400
    Processing by TwootsController#create as JS
      Parameters: {"utf8"=>"✓",
    "authenticity_token"=>"zXWgClq1GTM/rrB+p/othEJsGcdF61r+ERmm+FH7ojo=",
    "twoot"=>{"body"=>"twooting with a JS format hint\r\n"}, "commit"=>"Create
    Twoot"}
       (0.1ms)  BEGIN
      SQL (0.4ms)  INSERT INTO "twoots" ("body", "created_at", "updated_at") VALUES
    ($1, $2, $3) RETURNING "id"  [["body", "twooting with a JS format hint\r\n"],
    ["created_at", Sat, 21 Jun 2014 18:42:20 UTC +00:00], ["updated_at", Sat, 21
    Jun 2014 18:42:20 UTC +00:00]]
       (0.8ms)  COMMIT
    Completed 406 Not Acceptable in 2.9ms (ActiveRecord: 1.3ms)
```

OK, so, neat, this jives with the previous section about `respond_to`:
`TwootsController#create` can do different things based on the `Content-Type`
that is requested (be that XML, HTML, or JSON, etc.).  We also know that we can
request different formats by changing the `format` option to `form_for` (which
is analogous to changing the header in `curl`).

But something went wrong just now, we saw that the return status of this call
was an HTTP error code `406: Not Acceptable:`, that doesn't seem good.

```text
Completed 406 Not Acceptable in 2.9ms (ActiveRecord: 1.3ms)
```

How could we have made it acceptable?  By specifying, in the action, that we
would accept a _JavaScript_ call and by defining some JavaScript to respond with.
Let's do that.

    respond_to do |format|
      if @twoot.save
        format.html { redirect_to @twoot, notice: 'Twoot was successfully created.' }
        format.js { render json: @twoot, status: :created, location: @twoot }
      else
        format.html { render action: "new" }
        format.js { render json: @twoot.errors, status: :unprocessable_entity }
      end
    end

Here we say:

* Execute a block with a yielded value for `format`
* If the `Twoot` saves successfully...
  * then if the format type is `html` call the adjacent `Proc` which does a
    redirect.
  * If the type is `js` execute _that_ `Proc`.
* Otherwise follow a parallel "unhappy path"

So we add a `format.js` in the `TwootsController#create` and now when we
resubmit a new Twoot we see...

```text
    Started POST "/twoots.js" for 127.0.0.1 at 2014-06-21 14:59:56 -0400
    Processing by TwootsController#create as JS
      Parameters: {"utf8"=>"✓",
    "authenticity_token"=>"zXWgClq1GTM/rrB+p/othEJsGcdF61r+ERmm+FH7ojo=",
    "twoot"=>{"body"=>"gnu"}, "commit"=>"Create Twoot"}
       (0.1ms)  BEGIN
      SQL (1.0ms)  INSERT INTO "twoots" ("body", "created_at", "updated_at") VALUES
    ($1, $2, $3) RETURNING "id"  [["body", "gnu"], ["created_at", Sat, 21 Jun 2014
    19:00:02 UTC +00:00], ["updated_at", Sat, 21 Jun 2014 19:00:02 UTC +00:00]]
       (1.0ms)  COMMIT
    Completed 500 Internal Server Error in 8505.1ms

    ActionView::MissingTemplate (Missing template twoots/create, application/create
    with {:locale=>[:en], :formats=>[:js, :html], :handlers=>[:erb, :builder,
    :coffee]}. Searched in:
```

Rails told us that we're missing a `create.js.erb`.  We can create it!  In the
same way that you could create an _action_.html.erb which interprets `<% %>`
inside of HTML, you can do the _same_ thing were the `<% %>` are interpreted
into _JavaScript_.  

Let's make a ridiculously simple responder.

Add this responder in the `create` action.

`app/views/twoots/create.js.erb`

```javascript
alert("Your twoot about: '<%= @twoot.body %>' has been created!");
```

Let's try re-submitting the request and seeing what our server log tells us:

```text
    Started POST "/twoots.js" for 127.0.0.1 at 2014-06-21 15:02:57 -0400
    Processing by TwootsController#create as JS
      Parameters: {"utf8"=>"✓",
    "authenticity_token"=>"zXWgClq1GTM/rrB+p/othEJsGcdF61r+ERmm+FH7ojo=",
    "twoot"=>{"body"=>"gnu"}, "commit"=>"Create Twoot"}
       (0.1ms)  BEGIN
      SQL (1.0ms)  INSERT INTO "twoots" ("body", "created_at", "updated_at") VALUES
    ($1, $2, $3) RETURNING "id"  [["body", "gnu"], ["created_at", Sat, 21 Jun 2014
    19:03:01 UTC +00:00], ["updated_at", Sat, 21 Jun 2014 19:03:01 UTC +00:00]]
       (1.0ms)  COMMIT
      Rendered twoots/create.js.erb (1.1ms)
    Completed 200 OK in 3572.6ms (Views: 5.4ms | ActiveRecord: 2.2ms)
```

Awesome, 200 OK means success!  But what the browser shows, however is not
something quite that we expect.  We see interpolated JavaScript that _if_ it
had been executed would have put the body of the `Twoot` in an alert box.  To
test this theory, try copying and pasting that into your browser's developer
tools console.

Let's make a subtle change.  Let's remove the `format: :js` in `new.html.erb`
and turn it into `remote: true`.  This will still make a request in format JS
(check the server log to see it!) and will build that piece of JavaSCript code
again.  However, this time Rails will **automatically execute** the JavaScript.
Try it out and verify, but now an aysnchronous `POST` is fired (a.k.a "AJAX")
and the resulting JavaScript is automatically executed.

This is how, in the year 2005, Rails developers conceived of being able to work
around passing data from the front-end to the back end: they would just
interpolate the required data needed on the back end, and then have jQuery do
something clever with the returned data: _execute it as if it had been locally
created_.

### An Aside: Graceful Degradation

Rails developers of that era were also quite interested in "graceful
degradation."  What would happen if someone were to try the same operation on a
browser _without_ JavaScript?  The preferred behavior would be to "fall back"
to treating the request as if it had been made with an HTML `Content-Type` and
doing the "standard" redirect.  Using your developer browser tools you can turn
off JavaScript temporarily (in the Chrome's Console you'll see a "Gear" menu
that has a check box for temporarily disabling JavaScript).  Try submitting a
`Twoot` and instead of the `alert()` message just implemented, you should get
the usual `#show` template.

One last side effect of using jQuery to manage these requests and to do
graceful degradation is that after an AJAX event fires, Rails will trigger a
JavaScript event (`ajax:success`, `ajax:error`, or `ajax:failure`).

Let's set a listener on the document body so we can see that this event fires.

In `new.html.erb` let's add the following jQuery code:

```html
<script type="text/javascript" charset="utf-8">
  $("body").on("ajax:success", function(jQueryEventObject, data, textStatus) {
      alert('success was seen');
  });
</script>
```

Now when you create a `Twoot`, you'll get the alert from the rendered
`create.js.erb` _as well as_ the alert messsage that fires on `ajax:success`.

# Rails and JavaScript circa 2009

While the aforementioned techniques helped many Rails developers add whiz-bang
JavaScript animation tricks and avoid coming to know the richness of
JavaScript, these techniques were starting to reach a straining point.  Some
realized that we could use the Rails-specific AJAX events `ajax:success` and
render **partials** to have the server produce big hunks of DOM that could be
injected using basic jQuery.

This technique afforded Rails developers of the era the ability to write
minimal jQuery and yet stay in their comfort zone of Rails.

Let's add "create a new twoot" to be done within the index.  To do this we'll
modify `index.html.erb` to look like the following:

```html
<script type="text/javascript" charset="utf-8">
  $("body").on("ajax:success", function(e, data, textStatus) {
    $('#add-twoot').append(data);
  });
</script>

<h1>Listing twoots</h1>

<table>
  <tr>
    <th>Body</th>
    <th></th>
    <th></th>
    <th></th>
  </tr>

<% @twoots.each do |twoot| %>
  <tr>
    <td><%= twoot.body %></td>
    <td><%= link_to 'Show', twoot %></td>
    <td><%= link_to 'Edit', edit_twoot_path(twoot) %></td>
    <td><%= link_to 'Destroy', twoot, method: :delete, data: { confirm: 'Are you sure?' } %></td>
  </tr>
<% end %>

</table>
<div id='add-twoot'></div>

<br />

<%= link_to 'New Twoot', new_twoot_path, remote: true %>
```

By using this technique we can have the _server_ generate very large pieces of
DOM that we can inject with jQuery.  But there are still some issues.  If
we request the *actual* `new_twoot_path` action's template, we get the **Back**
element that's certainly appropriate to the `new` view but which doesn't make
sense when working with an inline AJAX call.  As a means to provide DRYer
access, let's extract out the form part from the view and then we'll focus only
only using *it* in the AJAX context.  How can we get _just_ the form part?
Fortunately, the Rails generators use a form partial within `new` and `edit`,
so we can choose to render _that_ partial instead of the template from the `new`
action.

What if we wanted to add Twooting from `/twoots`.  This is simple.

Add the following action to the `TwooterController`:

    def twoot_form
      @twoot = Twoot.new
      render :partial => 'form'
    end

And change the link on `#index`...

    <%= link_to 'New Twoot', new_twoot_form_path, remote: true %>

Now some rendered DOM partial will be injected because our `ajax:success`
listener is listening and waiting to inject that Rendered DOM via the following
code:

    <script type="text/javascript" charset="utf-8">
      $("body").on("ajax:success", function(e, data, textStatus) {
        $('#add-twoot')
          .append(data);
        // More UI mangling via jQuery to hide controls that should be hidden.
      });
    </script>

This is a great technique and is commonly used in many production Rails
codebases.  But since that time JavaScript has demanded more and more attention
and now many Rails developers are starting to conceive of Rails as a JSON API
factory which serves complex applications written in JavaScript or JavaScript
MVC libraries.

# Present Day

Since 2010 JavaScript has been undergoing a renaissance.  Thanks to Node.js,
Ember.js, Angular and other advances, JavaScript has come to be realized as a
*serious* programming language.

The present design pattern is to build JavaScript applications, in JavaScript
which are delivered by Rails (i.e. you create JavaScript files and have clients
load them).  Provided you have a methodology for structuring our JavaScript
sanely, we can have beautiful MVC architectures both in Rails and in
JavaScript: we need not make one language a codependent of the other.

This approach is what is taught in DBC Phases 2 and 3 presently.

Nevertheless, as you enter your career you may find some of these older
patterns of Rails + AJAX integration and you should be able to understand,
debug, and (hopefully!) refactor these into more current modalities.

Perhaps most confusing is that these techniques can be mixed: use a partial
to summon a form and then have it post to an action that has a JS responder
that uses jQuery to build up DOM nodes and inject them.  Given this complexity,
many developers have simply opted to build it in JavaScript

Obviously if Rails is merely a JSON API, this trend would point to the
extinction (or at least low barrier to leaving) Rails.  The Rails team wasn't
one to take that lightly and so they've improved Rails in two key ways as a
strategy against being obviated by the JS MVC frameworks: make Rails render
faster and make JavaScript easier to write on the server.  The technologies for
these goals are Turbolinks and CoffeeScript.  I will cover each below,
respectively.

# Present Day, alternate view: Turbolinks

While some have championed the idea of letting JavaScript frameworks (e.g.
Ember, Angular, etc.) handle front-end work and prefer to use the back-end as a
mere API, DHH and the rails-core team have presented an alternative that's
built into Rails: TurboLinks

[Turbolinks][] isn't so much a full-fledged front-end MVC as much as a means of
accelerating the page load within Rails.  The virtue of many of the JS MVCs is
that they can "we respond faster than Rails."  DHH responded with "Rails can
respond just as fast:" ergo, [Turbolinks][].  [Turbolinks][] gets its power by doing
fast DOM replacement of things that changed.  It's a bit like a fast,
pre-implemented version of the classic Rails AJAX paradigm with fetching
partials via AJAX and doing DOM replacement. To accomplish this it makes use of
JavaScript and `history.pushState()`.  To explain their exact use is outside
the scope of this document but can be examined further in the [Turbolinks][]
document.

To get a flavor for how it works, at a superficial level, visit:
`http://localhost:3000/twoots`.  Then, issue in the developer console: `
Turbolinks.visit('/twoots/2')`.  Observe:

1.  Fast page re-draw
2.  The server log shows that an HTML request was made

    Started GET "/twoots/2" for 127.0.0.1 at 2014-06-24 06:50:56 -0700
    Processing by TwootsController#show as HTML
      Parameters: {"id"=>"2"}
      Twoot Load (0.8ms)  SELECT  "twoots".* FROM "twoots"  WHERE "twoots"."id" =
    $1 LIMIT 1  [["id", 2]]
      Rendered twoots/show.html.erb within layouts/application (0.6ms)
    Completed 200 OK in 13ms (Views: 6.0ms | ActiveRecord: 0.8ms)

3.  The browser network tab in developer tools shows the detail about this
    request - basically that it requested the `#show` action's content but then
    used JavaScript to quickly cut out the old DOM and update it.

## Present Day, Alternative View: CoffeeScript

Many JavaScript MVC frameworks provide additional power to JavaScript or mutate
it into a class-based framework (v. prototypal inheritance model), provide
better iterator support, provide better inheritance solutions, etc.,  Rails
believes it provides a compelling counter offer by provided a native
CoffeeScript "transpiler" that, when deployed, will allow the parentheses free,
terse, CoffeeScript to become the verbose JavaScript we all know and love at
execution time.

CoffeeScript is built on the `_` or [underscore][] library which is essentially
an implementation of Ruby's Enumerable collection in JavaScript.  By providing
an intelligent transpiler and the horsepower of Enumerable, Rails feels that,
paired with [Turbolinks][], Rails _status quo_ is _plenty_ fast.

I find CoffeeScript to be beautiful and elegant.  Foremost it is a superior
whiteboarding technology.  Anyone who's whiteboarded pure JavaScript will tell
you that keeping track of parentheses stacks and writing the word `function`
over and over is not fun.  That said, when its transpiler goes to work it may
create JavaScript beyond your grasp and when you have a bug, you will have it
rendered, in the browser, as JavaScript.  I feel like CoffeeScript is a great
tool to have if you have a team which knows JS well.

[JSRails]: http://edgeguides.rubyonrails.org/working_with_javascript_in_rails.html
[111_respond_to]: http://apidock.com/rails/v1.1.1/ActionController/MimeResponds/InstanceMethods/respond_to
[Turbolinks]: https://github.com/rails/turbolinks
[scriptaculous]: http://script.aculo.us/
