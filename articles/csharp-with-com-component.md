---
title: "COMコンポーネントを使ったC#アプリケーションをコマンドプロンプトからビルドする"
emoji: "👮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "windows", "msbuild", "dotnet"]
published: true
---

結論としては、[`dotnet build` はCOMコンポーネントへの参照(`ResolveCOMReference`)に非対応](https://learn.microsoft.com/en-us/visualstudio/msbuild/resolvecomreference-task?view=vs-2022#msb4803-error)なので`msbuild`を使うのが正解。なぜこんなことに。

```
$ msbuild -t:clean -t:restore -t:build
```

とかすればいいです。`restore`タスクは`nuget restore`相当。
