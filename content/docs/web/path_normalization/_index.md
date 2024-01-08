---
weight: 1
bookFlatSection: true
title: "Path normalizationの挙動"
---

# Path normalizationの挙動
ファイルパスを正規化をする関数の挙動が与えられた引数により変わる。  
UnixとWindowsで若干挙動が異なるが今回はどちらでも影響がないためLinuxを用いて確認した。

確認したライブラリ
- Go(path/filepath)
- Python(os.path)
- NodeJS(Path)
- Ruby(pathname)


## 検証結果
相対パスと絶対パスで正規化した際にライブラリごとでどのような差が出るか確認した。  

- **Go**
    ```
    println(filepath.Clean("dir/../../hoge")) // relative
    println(filepath.Clean("/dir/../../hoge")) // absolute
    ```
    出力
    ```
    ../hoge
    /hoge
    ```

- **Python**
    ```
    print(os.path.normpath("dir/../../hoge")) // relative
    print(os.path.normpath("/dir/../../hoge")) // absolute
    ```
    出力
    ```
    ../hoge
    /hoge
    ```

- **JavaScript**
    ```
    console.log(path.normalize('dir/../../hoge')); //relative
    console.log(path.normalize('/dir/../../hoge')); //absolute
    ```
    出力
    ```
    ../hoge
    /hoge
    ```

- **Ruby**
    ```
    print(Pathname("dir/../../hoge").cleanpath()) // relative
    print(Pathname("/dir/../../hoge").cleanpath()) // absolute
    ```
    出力
    ```
    ../hoge
    /hoge
    ```

引数が相対パスか絶対パスかで出力が異なるので、開発者が想定していない出力結果を受け取る可能性がある。  
例として、開発者が「`filepath.Clean()`は「../」を取り除いてくれる」などの勘違いをした場合、パストラバーサルが発生する。

脆弱な例(Go)
```
path := req.FormValue("path")
uploadDir := filepath.Join("./upload", filepath.Clean(path)) 
...
```
