---
layout: post
author: Tony Schneider
title : Embracing Perspectives with the Rails Router
date  : 2020-06-01
---

One of my favorite parts of Rails is the [Rails router](https://guides.rubyonrails.org/routing.html).

Like anything, it has some gotchas.
For instance, the infamous guessing game of exactly what url helper gets generated that take a little while to get in your muscle memory...
But for the most part, once you wrap your head around it, it stays out of the way.

If you peek behind the curtain it has a number of interesting things going on.
Some of these include:

* Domain Specific Langage (DSL) for mapping routes to controller actions.
* Builder pattern constructing ruby objects that from the DSL.
* Registry for reflecting on said routes (e.g running `rails routes`)
* Establishing a code organization convention that cuts across the entire framework.

There's a ton to cover, and perhaps I will in future posts.
But for this post, I want to focus on the last item.

## A Problem of Perspectives

In my opinion, it's very easy for Rails developers to fall into the thought process of:

1. I have an ActiveRecord model named `Foo`
1. I should make an `FoosController`
1. All actions involving a `Foo` go into the `FoosController`

Ignoring the first point's ActiveRecord assumption, since I think it deserves its own post...

I would argue the guides (arguably rightly) point you in this direction.

This is harmless at first.
However, over time, this thought process can have huge impacts on the organization and discoverability of code in your application.

Let's take a look at how this is eventually problematic and one of the tools we have available to us to help remedy.

## A Humble Resource

```ruby
resource :jedis
```

Here I define a "jedis" resource.
As a result, Rails expects me to define a controller called `JedisController` located in `app/controllers/jedis_controller.rb`.

In addition, each controller action is now expected to have a "view" located in a directory that corresponds with the resource:

```
/app
  /views
    /jedis
      <action>.<extension(s)>
```

## Nested Resources

Eventually your app gains a requirement where it will grow some sort of nesting.
For instance, I imagine our jedis will need some lightsabres.

```ruby
resources :jedis, only: [:index] do
  resources :lightsabres, only: [:show]
end
```

If you're familiar with Rails, you'll know that this code expects you to define `JedisController` and `LightsabresController` controller classes.

In addition, we gain a new directory `app/views/lightsabres` to house our lightsabre views.
So far so good.
One with the force and all that noise.

## It's a Trap!

If you're anything like me, you'd agree that lightsabres are interesting in their own right.
Perhaps we want to list all the lightsabres independently of the jedi that happen to wield them?
Maybe they've been wielded by multiple jedi over time (tell me more!).

Goofy example aside, I've found wanting multiple perspectives of the same data to be a common pattern in large Rails apps.

Here's one way you might try to accomplish this:

```ruby
resources :jedis, only: [:index] do
  resources :lightsabres, only: [:show]
end
resources :lightsabres, only: [:index, :show]
```

This code expects the same controllers to be defined as before, but now we have two competing definitions of the `show` action.

```
/jedis/:jedi_id/lightsabres/:id
/jedis/lightsabres/:id
```

This is problematic because two routes now map to the same controller endpoint.
Last one wins.
**Confusion ensues.**

What I've seen most people do at this point is break out of the CRUD methods and define a one-off action in the controller.

With this decision, we begin to muddy the water of how our code is organized.

Our controller begins to wear multiple hats and we likely end up cramming many perspectives of a lightsabre into `app/views/lightsabres`.

## Embracing Many Perspectives

What can we do to highlight the important semantic difference between a lightsabre as an independent resource versus a lightsabre in the context of the jedi that wields it?

```ruby
resources :jedis, only: [:index] do
  resources :lightsabres, only: [:show], module: :jedis
end
resources :lightsabres, only: [:index, :show]
```

The only change I've made is that I've specified a module namespace of `:jedis` for the nested lightsabres resource.
I specifically chose `:jedis` because it's nested under the `:jedis` resource.

As a result, Rails now expects me to define **3** controllers:

* `JedisController` located at `app/controllers/jedis_controller.rb`
* `LightsabresController` located at `app/controllers/lightsabres_controller.rb`
* **`Jedis::LightsabresControllers`** located at `app/controllers/jedis/lightsabres_controllers.rb`

This allows both `LightsabresController` and `Jedis::LightsabresController` to define their own `show` action.

In addition, you now have an `app/controllers/jedis` directory and namespace to places all of the other resources that are from the perspective of an individual jedi.

As you may have guessed, this organizational pattern extends to the view as well.
This is great since the way you display a lightsabre by itself vs in the context of a specific jedi may vary greatly (this holds true when rendering JSON in an API scenario as well!)

This may seem like a minor change at first, but in my experience it can have a significant impact on how code is organized and discovered.

Personally, I'd recommend taking this approach for _every_ nested resource.
Interestingly, after taking this approach, I find little reason to break out of the default CRUD actions -- just define a new controller!

## A Parting Note

Over time, Rails has developed a specific interpretation of [REST](https://en.wikipedia.org/wiki/Representational_state_transfer).
Like anything, depending on your use case, it may not always suit your needs.

Personally, I've found the sweet spot to be server rendered applications (gasp!) and APIs with concrete use-cases.

However, regardless of how you use the rails router, don't fall into the trap of thinking your <span title="Read: Not limited to ActiveRecord">domain representations</span> must be 1-1 with your controller endpoints.

Try utilizing module namespacing to better organize your code!