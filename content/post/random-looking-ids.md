---
Description: How to generate random looking IDs - first try.
Keywords:
- crypto
- random
- uuid
Section: post
Slug: generating-random-looking-ids
Tags:
- uuid
- elixir
Title: Generating random looking IDs
Topics:
- Development
- Elixir
date: 2014-11-13
---


Generating random looking IDs
=============================

Introduction
------------

Quite often I find myself in a situation where I need a unique random looking IDs. The naive solution to this problem is to generate random IDs and memoize already issued ones to prevent duplicates. The question is can we do better?


Solution
--------

The soltution I am going to explore today is based on the use of `block ciphers`. Since their output is bijective (given same input IV and KEY) you will not have any collisions, unlike hashes.


## Elixir Implementation

```
defmodule CryptoSequence do
  defstruct key: nil, iv: nil, ctr: 0

  def new(args \\ []) do
    state = struct(__MODULE__, [
	    key: args[:key] || :crypto.rand_bytes(16),
	    iv: args[:iv] || :crypto.rand_bytes(16)
	  ])
    Stream.unfold(state, &generator/1)
  end

  defp generator(%__MODULE__{key: key, iv: iv, ctr: ctr} = state) do
    token = :crypto.block_encrypt(:aes_cfb128, key, iv, <<ctr :: 128 >>)
    {token, %{state | ctr: ctr + 1}}
  end

end
```


## Example of usage

### Return four tokens

```
sequence = CryptoSequence.new
sequence
  |> Stream.take(4)
  |> Enum.to_list
```

Let's try it
```
iex> sequence |> Stream.take(4) |> Enum.to_list
[<<24, 86, 163, 121, 95, 113, 126, 102, 200, 34, 35, 30, 58, 50, 200, 97>>,
 <<152, 122, 169, 74, 24, 168, 169, 92, 206, 91, 153, 8, 53, 194, 104, 104>>,
 <<75, 41, 106, 103, 124, 98, 251, 231, 91, 123, 213, 88, 169, 74, 12, 244>>,
 <<113, 137, 139, 169, 23, 166, 229, 213, 13, 20, 224, 141, 197, 91, 2, 13>>]
```


### Lazy filtering

```
sequence = CryptoSequence.new([size: 8])
sequence
  |> Stream.filter(fn(<<i :: 64>>) -> rem(i, 3) == 0 end)
  |> Stream.take(4)
  |> Enum.to_list
```

Let's try it
```
iex> sequence |> Stream.filter(fn(<<i :: 128>>) -> rem(i, 3) == 0 end) |> Stream.take(4) |> Enum.to_list
[<<75, 41, 106, 103, 124, 98, 251, 231, 91, 123, 213, 88, 169, 74, 12, 244>>,
 <<113, 137, 139, 169, 23, 166, 229, 213, 13, 20, 224, 141, 197, 91, 2, 13>>,
 <<155, 174, 200, 216, 52, 217, 97, 91, 201, 171, 209, 59, 226, 7, 91, 252>>,
 <<122, 227, 111, 91, 199, 52, 18, 116, 231, 60, 10, 124, 71, 93, 22, 127>>]
```


### Last checks

1. Let's check if generated IDs are unique

```
sequence = CryptoSequence.new
length(Stream.take(sequence, 255) |> Enum.to_list |> Enum.uniq) == 255
```

2. Let's check that stream instantiated using the same args emits same sequence

```
key = :crypto.rand_bytes(16)
iv = :crypto.rand_bytes(16)
sequence1 = CryptoSequence.new([key: key, iv: iv])
sequence2 = CryptoSequence.new([key: key, iv: iv])
(sequence1 |> Stream.take(4) |> Enum.to_list) == (sequence2 |> Stream.take(4) |> Enum.to_list)
```

```
Iv = crypto:rand_bytes(16).
Key = crypto:rand_bytes(16).
[base64:encode(crypto:block_encrypt(aes_cfb128, Key, Iv, <<I:32>>)) || I <- lists:seq(1, 255)].
```

### Note to myself

I need to analyse if the solution is subject to `padding oracle attack`. I would need to look at other bijective functions if it is.
