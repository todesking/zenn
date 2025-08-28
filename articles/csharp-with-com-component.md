---
title: "COMコンポーネントを使ったC#アプリケーションをコマンドプロンプトからビルドする"
emoji: "👮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "windows", "msbuild", "dotnet"]
published: true
---

```shellsession
$ dotnet build
Restore complete (0.3s)
  TestProject failed with 1 error(s) (0.0s)
      /usr/local/share/dotnet/sdk/9.0.304/Microsoft.Common.CurrentVersion.targets(3081,5): error
 MSB4803: The task "ResolveComReference" is not supported on the .NET Core version of MSBuild.
  Please use the .NET Framework version of MSBuild. See https://aka.ms/msbuild/MSB4803 for further details.
```

結論としては、[`dotnet build` はCOMコンポーネントへの参照(`ResolveCOMReference`)に非対応](https://learn.microsoft.com/en-us/visualstudio/msbuild/resolvecomreference-task?view=vs-2022#msb4803-error)なので`msbuild`を使うのが正解。なぜこんなことに。

```
$ msbuild -t:clean -t:restore -t:build
```

とかすればいいです。`restore`タスクは`nuget restore`相当。

`restore`しない場合、`clean`後に`build`した際エラーが出ます:

```
(ResolvePackageAssets target) ->
C:\Program Files\dotnet\sdk\9.0.304\Sdks\Microsoft.NET.Sdk\targets\Microsoft.PackageDependencyResolution.targets(266,5):
  error NETSDK1064: Package AWSSDK.S3, version 4.0.6.8 was not found. It might have been deleted since NuGet restore.
  Otherwise, NuGet restore might have only partially completed, which might have been due to maximum path length restrictions.
```
