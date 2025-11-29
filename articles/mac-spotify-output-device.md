---
title: "Spotify for Macの出力先デバイスを変更する"
emoji: "🎵"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Spotify", "Mac"]
published: true
---

Spotify for Macに出力デバイス設定機能はなく、[2014年から要望されてる](https://community.spotify.com/t5/Desktop-Mac/Mac-Change-audio-output-of-Spotify-App-ONLY/m-p/1006038#M274494)けど音沙汰なしという状態、macOSにもアプリごとの出力デバイスを設定する機能はなし、有償のサードパーティツール[SoundSource](https://www.rogueamoeba.com/soundsource/)を使用することでアプリごとの出力デバイス設定を制御できるが45USDする(実装にかなりの技術が必要らしく、これ以外の選択肢は発見できなかった)。なぜこんな仕打ちを受けるのかというと、不自由で邪悪なOS上で不自由で邪悪なソフトウェアを動かした報いである。

唯一手軽な方法としては、librespotというCLIアプリがある。これを使うとSpotify Connectを使用してSpotifyの音楽を再生できて、操作はSpotifyアプリ上で可能。

```shellsession
$ brew install librespot
```

して、

```shellsession
$ librespot --device "(出力先デバイス名)" \
  --initial-volume 100 \
  -j \
  --cache ~/.cache/librespot \
  --system-cache ~/.cache/librespot \
  --bitrate 320
```

で起動。`-j`オプションでoAuth認証使用。キャッシュディレクトリを指定しないと認証情報も保存されないので注意。この状態で、Spotifyアプリからデバイスを選択して操作可能になる。

`--bitrate`オプションでビットレートを選べるが、losslessには非対応なので注意。[技術的には対応可能だがSpotifyの圧力で実装不能になった](https://github.com/librespot-org/librespot/issues/1583#issuecomment-3499343020)という経緯がありFUCK DRMということになるが、これもまた不自由で邪悪なサービスに依存した報いであろう(おわり)。
