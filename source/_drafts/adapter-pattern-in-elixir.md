---
layout: post
title: Adapter pattern with Elixir
tags:
  - elixir
---

Lately I came accros an usual problem. How can I use a fake API for development and test and the real one for
production?

This example will show my implementation using the yify subtitles API as example.

## Implement the backbone

First, we need to define our specification.

In order to fetch subtitles, one parameter is required: the IMDb id.

If we want we can also filter to search only the languages we are interested in (i.e: english, french,..).

The return value will be a struct containing the IMDb id and a list of languages available with their URLs to the
subtitles.

Basically we need an Elixir module which implements the search method:

```elixir
# lib/yify_subtitle.ex
defmodule YifySubtitle do
  defstruct imdb_id: "", languages: []

  def search(imdb_id, languages \\ []) do
    # API call here
  end
end
```

## Implement the fake

With this basic template done we can start to implement the fake implementation.

Arbitrarily I decided the fake API would handle only 3 languages: english, french and spanish.

It means, if we don't set any languages, we should return all these languages. It also means, if we set the languages
we want, we should return only these languages (if included in the available languages).

```elixir
# lib/yify_subtitle/adapter/in_memory.ex
defmodule YifySubtitle.Adapter.InMemory do
  @url       "http://api.fake.com/"
  @languages [:english, :french, :spanish]

  def search(imdb_id, []), do: search(imdb_id, @languages)
  def search(imdb_id, languages) do
    languages
    |> filter_languages
    |> formalize_response(imdb_id)
    |> Enum.to_list
  end

  defp filter_languages(languages) do
    languages
    |> Stream.filter(&(Enum.member?(@languages, &1)))
  end

  defp formalize_response(languages, imdb_id) do
    languages
    |> Stream.map(&({&1, ["#{@url}#{imdb_id}/#{&1}.zip"]}))
  end
end

# test/adapter/in_memory_test.exs
defmodule YifySubtitle.Adapter.InMemoryTest do
  use ExUnit.Case

  test "search returns list of all available subtitles" do
    expected = [
      english: ["http://api.yifysubtitles.com/subtitle-api/tt0133093/english.zip"],
      french:  ["http://api.yifysubtitles.com/subtitle-api/tt0133093/french.zip"],
      spanish: ["http://api.yifysubtitles.com/subtitle-api/tt0133093/spanish.zip"]
    ]

    assert expected == YifySubtitle.Adapter.InMemory.search("tt0133093", [])
  end

  test "search returns only list of available subtitles" do
    expected = [
      english: ["http://api.yifysubtitles.com/subtitle-api/tt0133093/english.zip"],
    ]

    assert expected == YifySubtitle.Adapter.InMemory.search("tt0133093", [:english, :dutch])
  end
end
```

Now we have a fake adapter, we can update our entry point to use it:

```elixir
# lib/yify_subtitle.ex
defmodule YifySubtitle do
  defstruct imdb_id: "", languages: []

  def search(imdb_id, languages \\ []) do
    %YifySubtitle{
      imdb_id:   imdb_id,
      languages: YifySubtitle.Adapter.InMemory.search(imdb_id, languages)
    }
  end
end

# test/yify_subtitle_test.exs
defmodule YifySubtitleTest do
  use ExUnit.Case

  test "search returns list of all available subtitles" do
    expected = %YifySubtitle{
      imdb_id:   "tt0133093",
      languages: [
        english: ["http://api.yifysubtitles.com/subtitle-api/tt0133093/english.zip"],
        french:  ["http://api.yifysubtitles.com/subtitle-api/tt0133093/french.zip"],
        spanish: ["http://api.yifysubtitles.com/subtitle-api/tt0133093/spanish.zip"]
      ]
    }

    assert expected == YifySubtitle.search("tt0133093")
  end

  test "search returns only list of available subtitles" do
    expected = %YifySubtitle{
      imdb_id:   "tt0133093",
      languages: [
        english: ["http://api.yifysubtitles.com/subtitle-api/tt0133093/english.zip"],
      ]
    }

    assert expected == YifySubtitle.search("tt0133093", [:english, :dutch])
  end
end
```

We can now run the tests to see if everything works fine.

```bash
% mix test
....

Finished in 0.05 seconds (0.05s on load, 0.00s on tests)
4 tests, 0 failures
```

Excellent! but we don't have any real API call yet. Moreover, the interface hard-codes the call to the in memory adapter.

## Implement the real adapter

Now we have a fake implementation and we are sure everything works well. We can implement the real service following the
same contract.

First we need to add 3 dependencies to our project:

* [HTTPoison](https://github.com/edgurgel/httpoison): an http library
* [Poison](https://github.com/devinus/poison): a JSON parsing library
* [mock](https://github.com/jjh42/mock): a mocking library, useful to mock HTTP calls

```elixir
# mix.exs
defmodule YifySubtitle.Mixfile do
  use Mix.Project

  # omitted code..

  def application do
    [applications: [:httpoison, :logger, :poison]]
  end

  defp deps do
    [
      {:httpoison, "~> 0.6"},
      {:mock,      "~> 0.1.0"},
      {:poison,    "~> 1.4.0", only: :test}
    ]
  end
end
```

Then, we can install the dependencies with `mix deps.get`.

We are finally ready to tackle the hard part: The implementation of the real adapter.

```elixir
# lib/yify_subtitle/adapter/api.ex
defmodule YifySubtitle.Adapter.API do
  @url "http://api.yifysubtitles.com"

  def search(imdb_id, languages) do
    imdb_id
    |> build_url
    |> fetch_from_api
    |> transform_http_response_to_data
    |> filter_languages(languages)
    |> formalize_response
    |> Enum.to_list
  end

  defp build_url(imdb_id) do
    "#{@url}/subs/#{imdb_id}"
  end

  defp extract_urls_from_subtitle_infos(subtitles_infos) do
    subtitles_infos
    |> Enum.map(fn(%{"url" => path}) -> @url <> path end)
  end

  defp fetch_from_api(url) do
    case HTTPoison.get!(url) do
      %HTTPoison.Response{status_code: 200, body: body} -> Poison.Parser.parse!(body)
    end
  end

  defp filter_languages(subtitles, []), do: subtitles
  defp filter_languages(subtitles, expected_languages) do
    subtitles
    |> Stream.filter(fn({language, _}) -> Enum.member?(expected_languages, String.to_atom(language)) end)
  end

  defp formalize_response(subtitles) do
    subtitles
    |> Stream.map(fn({language, subtitles_infos}) -> {String.to_atom(language), extract_urls_from_subtitle_infos(subtitles_infos)} end)
  end

  defp transform_http_response_to_data(response) do
    case response do
      %{"success" => true, "subs" => results} ->
        [{_, subtitles}] = results |> Map.to_list
        subtitles
      %{"success" => true} -> %{}
    end
  end
end

# test/adapter/api_test.exs
defmodule YifySubtitle.Adapter.APITest do
  use ExUnit.Case, async: false

  import Mock

  test "search returns list of all available subtitles" do
    with_mock HTTPoison, [
      get!: fn("http://api.yifysubtitles.com/subs/tt0133093") ->
              %HTTPoison.Response{status_code: 200, body: File.read!("test/adapter/fixtures/tt0133093.json")}
            end
    ] do
      expected = [
        english: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-742.zip", "http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-39316.zip"],
        french:  ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-24107.zip"],
        spanish: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-4242.zip"]
      ]

      assert expected == YifySubtitle.Adapter.API.search("tt0133093", [])
    end
  end

  test "search returns only list of available subtitles" do
    with_mock HTTPoison, [
      get!: fn("http://api.yifysubtitles.com/subs/tt0133093") ->
              %HTTPoison.Response{status_code: 200, body: File.read!("test/adapter/fixtures/tt0133093.json")}
            end
    ] do
      expected = [
        english: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-742.zip", "http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-39316.zip"]
      ]

      assert expected == YifySubtitle.Adapter.API.search("tt0133093", [:english, :dutch])
    end
  end
end
```

We can now run the test again with the new API implementation:

```bash
 % mix test
......

Finished in 0.1 seconds (0.06s on load, 0.09s on tests)
6 tests, 0 failures

Randomized with seed 957004
```

Perfect, we now have two implementations! We are closer. The only remaining problem is to be able to switch from one
adapter to another.

## Swap the adapter

Now we are quite confident about our implementations, we need one thing: a way to switch from one adapter to another.

Hopefully, Elixir has our cover! It comes with a way to configure default environment values for our applications.

First, we need to improve our config file to be able to handle multiple environments:

```elixir
# config/config.exs
use Mix.Config

import_config "#{Mix.env}.exs"
```

With this modification our config file will load a file, based on our current environment, when we compile the project.

Next, we have to set which adapter we want to use in each environment. That's pretty straightforward:

```elixir
# config/dev.exs
use Mix.Config

config :yify_subtitle, :adapter, YifySubtitle.Adapter.InMemory

# config/prod.exs
use Mix.Config

config :yify_subtitle, :adapter, YifySubtitle.Adapter.API

# config/test.exs
use Mix.Config

config :yify_subtitle, :adapter, YifySubtitle.Adapter.InMemory
```

And, finally, we have to update the entry point to use the good adapter:

```elixir
defmodule YifySubtitle do
  defstruct imdb_id: "", languages: []

  def search(imdb_id, languages \\ []) do
    %YifySubtitle{
      imdb_id:   imdb_id,
      languages: adapter.search(imdb_id, languages)
    }
  end

  defp adapter, do: Application.get_env(:yify_subtitle, :adapter)
end
```

We can just test it in console, just to be sure it works fine:

```bash
% iex -S mix
iex(1)> YifySubtitle.search("tt0133093")
%YifySubtitle{imdb_id: "tt0133093",
 languages: [english: ["http://api.fake.com/tt0133093/english.zip"],
  french: ["http://api.fake.com/tt0133093/french.zip"],
  spanish: ["http://api.fake.com/tt0133093/spanish.zip"]]}

% MIX_ENV=prod iex -S mix
iex(1)> YifySubtitle.search("tt0133093")
%YifySubtitle{imdb_id: "tt0133093",
 languages: [arabic: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-741.zip"],
  bengali: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-23870.zip",
   "http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-4241.zip"],
  "brazilian-portuguese": ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-9841.zip"],
  bulgarian: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-40238.zip"],
  chinese: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-18505.zip",
   "http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-52083.zip",
   "http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-17090.zip"],
  croatian: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-49123.zip"],
  dutch: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-7566.zip"],
  english: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-742.zip",
   "http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-39316.zip",
   "http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-52597.zip",
   "http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-40402.zip",
   "http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-52598.zip"],
  "farsi-persian": ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-743.zip"],
  french: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-24107.zip"],
  greek: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-9816.zip"],
  hebrew: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-10996.zip"],
  hungarian: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-42520.zip"],
  indonesian: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-32218.zip"],
  italian: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-12961.zip"],
  korean: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-10418.zip",
   "http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-13211.zip"],
  portuguese: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-11078.zip"],
  romanian: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-17462.zip"],
  russian: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-41611.zip",
   "http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-41647.zip"],
  serbian: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-10883.zip"],
  slovenian: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-15236.zip"],
  spanish: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-4242.zip"],
  turkish: ["http://api.yifysubtitles.com/subtitle-api/the-matrix-yify-47800.zip"]]}
```

That's all - finally, we have an application implementing an adapter pattern. That's definitively a basic implementation
but it works pretty well.

So what's next? The next things to work on could be:

* how to use the adapter pattern with OTP compliant adapters
* how to use the exact same tests between the fake and the real service, to be more confident
