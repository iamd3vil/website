+++
title = "HMAC in Elixir and Python"
description = "HMAC in Elixir and Python"
date = 2016-02-23
slug = "hmac_in_elixir"

[taxonomies]
tags = ["elixir", "erlang", "hmac", "crypto", "python"]
+++

Recently I had to implement HMAC SHA1 using Elixir. It's pretty simple in Python. Here's the code in Python3.

```python
import hashlib
import hmac

secret_key = b"secret-key"
text = b"This is a secret"

hmac_sha1 = hmac.HMAC(key=secret_key, msg=text, digestmod=hashlib.sha1).hexdigest()
print("HMAC is {}".format(hmac_sha1)) # HMAC is 08dc7014b3e778a44af52ea7a16a973a9b48f0dd
```

I wanted to do the same, but couldn't find any resource so I went into the Erlang's `crypto` module and it has a `hmac/3` function which does the same. This is how we can use the erlang module to create a HMAC SHA1 in Elixir.

```elixir
secret_key = "secret-key"
text = "This is a secret"

hmac = :crypto.hmac(:sha, secret_key, text)
       |> Base.encode16
       |> String.downcase
IO.puts "HMAC is #{hmac}" # HMAC is 08dc7014b3e778a44af52ea7a16a973a9b48f0dd
```

In the `hmac/3` you can also pass `:md5`, `:sha224`, `:sha256`, `:sha384`, `:sha512` for different algorithms. Erlang's `crypto` module is awesome and pretty powerful. This is where, I very much like Elixir and Erlang's interoperability.
