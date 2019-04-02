---
layout: post
title: "Ignoring the return value"
date: 2018-05-03
categories: blog
tags: elixir

image: "/assets/images/posts/elixir-underscore.jpg"
---

Two weeks ago [José Valim](https://github.com/josevalim){:target="_blank"} created a [Pull Request](https://github.com/phoenixframework/phoenix/pull/2861){:target="_blank"} in the [Phoenix project](https://github.com/phoenixframework/phoenix){:target="_blank"}. One of the things that stood out to me when I was taking a look at it was [this line](https://github.com/phoenixframework/phoenix/pull/2861/files#diff-52a5a0836514e00b4177376ab40cbe23R243){:target="_blank"}.

Here, José is pattern matching to underscore the result value of the call to `Process.monitor/1`, which at first looks like an unnecessary match, due to the fact that the result will be ignored.

The full function block looks something like this:

{% highlight elixir %}
def init({socket, auth_payload, parent, ref}) do
  _ = Process.monitor(socket.transport_pid)
  %{channel: channel, topic: topic} = socket
  socket = %{socket | channel_pid: self()}

  case channel.join(topic, auth_payload, socket) do
    {:ok, socket} ->
      init(socket, %{}, parent, ref)
    {:ok, reply, socket} ->
      init(socket, reply, parent, ref)
    {:error, reply} ->
      send(parent, {ref, reply})
      :ignore
    other ->
      raise """
      channel #{inspect socket.channel}.join/3 is expected to return one of:
          {:ok, Socket.t} |
          {:ok, reply :: map, Socket.t} |
          {:error, reply :: map}
      got #{inspect other}
      """
  end
end
{% endhighlight %}

As [he explains](https://github.com/phoenixframework/phoenix/pull/2861/files#r183449212){:target="_blank"}:

> Yes, we usually do it to say "i know this returns something meaningful, but i am sure i don't need it".

This seems to be a good convention to show we won't use the returned expression, so I will take it into account to use it in the future.
