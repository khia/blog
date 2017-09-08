---
lastmod: "2017-05-17"
date:        "2013-04-21"
title:       "Elixir binding for lager"
description: "Implements a simple wrapper over https://github.com/basho/lager."
tags:        [ "Development", "Elixir" ]
topics:      [ "Development", "Elixir" ]
slug:        "exlager"
project_url: "http://github.com/khia/exlager"
---

# ExLager

Embeds logging calls to ExLager into a module if currently configured logging level is less or equal
than severity of a call. Otherwise no code is emmited. Therefore it doesn't have any negative impact
on performance of a production system when you configure error level even if you have tons of
debug messages.
