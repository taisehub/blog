---
weight: 1
bookFlatSection: true
title: "BeautifulSoupのパーサについて"
---

# BeautifulSoupのパーサについて

HTMLパーサとブラウザ間でHTMLの解釈に差があればXSSが発生するリスクがあるよという話をします。  
開発者の皆さんも自前でHTMLパーサを実装することはほとんどないと思うので、各々が信頼するHTMLパーサを利用することになるかと思います。  
有名なパーサーでもHTMLの解釈はとても複雑なため想定外の挙動を引き起こす可能性があります。  
実際、とあるHTMLパーサの不適切な解釈について脆弱性を報告しました。(まだ開示されていないので今回は触れません。)  

今日はBeautifulSoupの話をします。

## BeautifulSoupの使い方に注意　
HTMLはタグの種類によって解釈が変わります。  
`<script>`タグもその一つになります。

`<script>`タグの中身はJavaScriptとして扱われるので、`<script>`タグの中身がHTMLの構造に影響することは直感的に無いように思ますが、実はあります。  
`<script>`タグの中のJavaScriptは状態を持ち、その状態により`</script>`が`<script>`タグの閉じタグとして扱われるかが変わります。

この記事で`<script>`タグ内の状態については深く触れませんが、例として以下のような二つのHTMLを考えます。
分かりやすいように状態をコメントしています。

```html
<script> 
    // script data state
        <script>
        </script>
        <script>
</script>
```

```html
<script> 
    // script data state
    <!-- // script data escaped state
        <script>
          // script double data escaped state
        </script>
        <script>
    -->
    // script data state
</script>
```

一つ目のHTMLをブラウザで表示すると`<script>...</script>`が二つ表示されると思います。
一方で二つ目のHTMLをブラウザで表示すると`<script>`タグの中に文字列として`<script>...</script>`が含まれていると思います。
ネストした`</script>`タグはscript data double escaped stateなので`<script>`タグは閉じられません。

BeautifulSoupの`html.parser`はこの状態を解釈しないため、一つ目の`</script>`タグが最初の`<script>`タグを閉じてしまいます。  
```python3
from bs4 import BeautifulSoup

soup = BeautifulSoup("<script> <!-- <script></script> --> </script>", 'html.parser')
print(soup) # 出力: <script> <!-- <script></script> --&gt;
```

これは、BeautifulSoupのドキュメントでも述べられています。

{{< figure src="./beautifulsoup_parsers.png"  class="center" >}}

```python3
from bs4 import BeautifulSoup

soup = BeautifulSoup("<script> <!-- <script></script> --> </script>", 'html5lib')
print(soup) # 出力: <script> <!-- <script></script> --> </script>
```

利用するHTMLパーサの違いは考慮が漏れやすいと思いますが、 ユーザ入力などでHTMLを受け付ける場合はより厳密な`html5lib`を使った方が良いです。

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
