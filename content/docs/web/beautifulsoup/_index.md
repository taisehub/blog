---
weight: 1
bookFlatSection: true
title: "BeautifulSoupの挙動"
---

HTMLパーサの挙動を調査していたところ、BeautifulSoupの怪しい挙動を発見したのでメモ。

## 詳細

```python3
from bs4 import BeautifulSoup

soup = BeautifulSoup("<script> <!-- <script></script> --> </script>", 'html.parser')
print(soup)
print(soup.find_all("a"))

// 出力: <script> <!-- <script></script> --&gt;
```

何がまずいのかというと、scriptタグ内で状態の遷移が行われていない。
これをブラウザで読むと以下のようになるはず。

```html
<script> 
    <!--
        <script></script>
    -->
</script>
```

## より安全な実装
`html5lib`を使おう。

```python3
from bs4 import BeautifulSoup

soup = BeautifulSoup("<script> <!-- <script></script> --> </script>", 'html5lib')
print(soup)
print(soup.find_all("a"))

// 出力: <script> <!-- <script></script> --&gt;
```

## おまけ

調査の過程で見つかった自分が知らなかったことのメモ


### script data double escape stateへの遷移について
[golang.org/x/net/html](https://pkg.go.dev/golang.org/x/net/html)`script data double escape state`(正式名称は違う気がするけどメモなのでOK)に遷移する際に利用する`<script>`タグが、実は最後の`>`が必要ないことに気づいた。
`>`の代わりに、以下のいずれかでも遷移する
```
' ', '\n', '\r', '\t', '\f', '/', '>'
```

`data double escape state`に遷移する例
```html
<script>
    <!--
        <script/
        </script>
    -->
</script>
```
