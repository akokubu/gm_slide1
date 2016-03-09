## Elixir & Phoenix
![Elixirロゴ](images/elixir.jpeg "Elixir")

![Phoenixロゴ](images/phoenix.png "Phoenix")

***
## WEB+DB
* vol.88-89
* 伊藤 直也さんの記事
---
![伊藤直也](images/naoyaito.jpeg "naoyaito")
***
ある会社で、新しい機能を開発することになって、
Elixirを使いたいと提案したら・・・
---
新しい言語か。2年後には消えてるかもしれない・・・<br>
本番で使うのはやめてくれ
***
## Elixirの特徴
* 名前がかっこいい
* Ruby on Railsのコアメンバーが作ってる
* 今流行りの関数型言語
* 速い
* 並行処理に優れている
* Erlang VM上で動く


<b style="color: red;">Elixirとは、Erlangをよりわかりやすく書ける</b>

---
## Erlangとは
知名度はそんなに高くない。<br/>
でも、使われているところでは使われているらしい。

* Ericsson Languageの略。
※Ericsonは電話会社
* 電話の交換器などに使われている
* 大量の並列処理
* 耐障害性
* 言語の拡張が可能 
* Erlangの歴史は長く、安定している。
* ニコニコやTwitter、LINEでも使われている。

<b style="color: red;">要はErlangすごい</b>

***

***
## Elixirのコード


```
defmodule WeatherClient do
  def fetch_json(url) do
    HTTPoison.start
    res = HTTPoison.get!(url)
    Poison.decode!(res.body)
  end

  def get do
    %{"weather" => weather} = 
      fetch_json("http://api.openweathermap.org/data/2.5/weather?q=Tokyo,jp")
    weather
  end

  Enum.each WeatherClient.get, fn(w) ->
    IO.puts w["main"]
  end
end
```

---

### データ型は不変(Immutable)

```
a = [1, 2, 3]

iex> List.delete_at(a, 0)
[2, 3]

iex> a
[1, 2, 3]
```
* 1が削除された新しいリストが生成される。
* 元のリストは変更されない。

→ 副作用がない。関数型言語の特徴

---

## パターンマッチング

```
iex> {a, b, c} = {:hello, "world", 42}

iex> a
:hello

iex> b
"world"

iex> c
42
```


```

%{status: matched_status, body: matched_body} = %{status: 200, body: "V8!"}

```
これで、matched_statusの中身は`200`が入っていて、matched_bodyには`V8!`が格納されている。


変数だけでなく条件や関数でも使われる。


---
パターンマッチングによる処理の振り分け

```
result = File.read("/etc/hosts") do
    {:ok, res} -> res
    {:error, :enoent} -> "Not Found"
    {:error, :eaccess} -> "FOrbidden"
    _ -> "?"
end
```
File.readの戻り値によって処理を分岐。

if-elseで分岐よりもわかりやすい。
---

関数でも

```
defp process_response(%{status_code: 301, headers: headers}) do
  redirect headers
end

defp process_response(%{status_code: 302, headers: headers}) do
  redirect headers
end

defp process_response(%{status_code: 200, body: body}) do
  IO.puts body
end

defp redirect(headers) do
  headers |> Dict.get(:Location) |> get
end
```
if-elseで分岐させるよりもわかりやすい。

慣れてくれば・・・
---
配列の場合

```
iex> [h|t] =  [3.41, :pie, "Apple"]
[3.41, :pie, "Apple"]

iex> h
3.41

iex> t
[:pie, "Apple"]
```
<pre>
パイプ演算子を使って、headとtailに分ける
head: リストの先頭の要素
tail: 残りの要素
※ リストから1つずつ取り出して処理する時など
</pre>

---

## forやwhileがない。
だったら、再帰使えばいい。

ちょっと何言ってるかわからない。

---

パターンマッチングと再帰によるリスト操作

```
defmodule MyList do
  def each(h|t], f) do
    f.(h)
    each(tail, f)
  end

  def each([], _), do: nil
end

MyList.each [1, 2, 3, 4, 5], fn (x) -> IO.puts x end
```

* [h|t]で配列の先頭と残りに分けて、先頭を処理。
* 残りを再帰で処理。

---
こんなのをいちいち書いてる人はいません。

標準ライブラリとして提供されています。

```

Enum.map [1, 2, 3, 4, 5], fn (x) -> IO.puts x end

```
リストと処理する関数を渡せばやってくれる。

※ Enum.filter、Enum.sumとか色々ある。

要するにunderscoreみたいな。


---
## パイプライン演算子

```

body
|> Poison.decode!
|> extract_weather

```

前の関数の戻り値が次の関数の第一引数になる。

```
decodedBody = Poison.decode!(body)
weather = extract_weather(decodedBody)

または

extract_weather(Poison.decode!(body))
```

---

```
defmodule Weather do
    def get do
        HTTPoison.start
        HTTPoison.get!("http://api.openweathermap.org/data/2.5/weather?q=Tokyo,jp")
        |> process_response
    end

    defp process_response(%{status_code: 200, body: body}) do
        body
        |> Poison.decode!
        |> extract_weather
    end

    defp process_response(%{status_code: 302, headers: headers}) do
        redirect headers
    end  

    def extract_weather(%{"weather" => weather}), do: weather

end

Enum.each Weather.get, fn(w) ->
    IO.puts w["main"]
end
```

Q: status_code = 404だったら？
---

A: クラッシュします。

---

Erlangではエラー処理をがんばるのではなく、

クラッシュさせてしまうのがお作法。


<strong>Let it Crash!</strong>

訳：クラッシュさせちまいな！

---

クラッシュは別のプロセスで監視しておき、<br/>エラーハンドリングは監視プロセスで行う。

###ロジックとエラーハンドリングの分離
* コードの保守性向上
* 耐障害性の向上

使い捨てられるくらいにErlangの<br/>プロセス生成コストは安い。

---
Erlang OTP（Open Telecom Platform）<br/>
こいつがすべてやってくれる。Erlangすごい。

こいつの上に構築された<br>アプリケーションフレームワークが、

![Phoenixロゴ](images/phoenix.png "Phoenix")

---
ほぼ、Ruby onn Railsです。

routes

```
get "/", PageController, :index
get "/hello/:name", PageController, :hello
```

controller

```
def hello(conn, %{"name": name}) do
  render conn, "hello.html" name: name
end
```

view

```
<h2>Hello world, from <%= @name %>!</h2>
```
---
![Phoenixロゴ](images/phoenix.png "Phoenix")

The Road to 2 Million Websocket Connections in Phoenix

1台で200万アクセスくらいさばけるらしい・・・

---
# Thanks!

