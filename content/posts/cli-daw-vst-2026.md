---
title: "CLIで動くDAWはあるのか？VST対応ヘッドレスDAWを総調査する"
slug: "cli-daw-vst-2026"
date: 2026-07-07
draft: false
categories: ["音楽制作"]
tags: ["DAW", "VST", "CLI", "DawDreamer", "Renoise", "ヘッドレス", "Csound"]
description: "ターミナルからVSTをロードしてレンダリングできるのか？CLIベースDAWの選択肢を徹底調査"
---

## 背景

ふとしたきっかけで「CLI（コマンドライン）だけで動くDAWってあるの？」という疑問が湧いた。具体的には「ターミナルからVSTプラグインをロードして、MIDIファイルを流し込み、オーディオをレンダリングしたい」というユースケースだ。

理由はいくつかある：

- CI/CDパイプラインで自動ミックスダウンしたい
- ヘッドレスサーバーでバッチ処理したい
- エディタ（Vim/Neovim）から離れずに音楽制作したい
- Dockerコンテナ内で完結する音楽生成パイプラインを作りたい

GUIが前提のAbleton LiveやFL Studioではこうはいかない。そこで、CLIベース／ヘッドレス運用が可能なDAW的ツールを徹底的に調べてみた。

## 調査結果：CLIでVSTを動かす4つの選択肢

### 1位：DawDreamer — Pythonで書けるヘッドレスDAW

もっとも現実的で柔軟性が高いのが **DawDreamer** だ。JUCEベースのDAWエンジンをPythonから操作できるライブラリで、ほぼ完全な「Pythonで書けるDAW」と言ってよい。

**基本スペック**

- GitHub: [DBraun/DawDreamer](https://github.com/DBraun/DawDreamer)（⭐1.3k）
- VST2 / VST3 両対応（インストゥルメント＆エフェクト）
- `pip install dawdreamer` でインストール可能
- Windows / macOS / Linux / Google Colab 対応
- 完全ヘッドレス → DockerやCIでも使える

**サンプルコード**

VSTインストゥルメントをロードしてMIDIシーケンスをレンダリングする基本的な流れは以下の通り：

```python
import dawdreamer as daw
from scipy.io import wavfile

# エンジンを作成（サンプルレート、バッファサイズ）
engine = daw.RenderEngine(44100, 512)

# VST3プラグインをロード（パスは環境に合わせて変更）
synth = engine.make_plugin_processor("synth", "path/to/your/synth.vst3")

# MIDIノートを配置（ノート番号、ベロシティ、開始秒、長さ秒）
synth.add_midi_note(60, 100, 0.0, 2.0)  # C4
synth.add_midi_note(64, 100, 2.0, 2.0)  # E4
synth.add_midi_note(67, 100, 4.0, 2.0)  # G4

# グラフを組む（VST → 出力）
engine.load_graph([
    (synth, []),
])

# レンダリング
engine.render(8.0)  # 8秒間

# オーディオデータを取得（numpy配列）
audio = engine.get_audio()

# WAVファイルに書き出し
from scipy.io import wavfile
wavfile.write("output.wav", 44100, audio.T)
```

パラメータオートメーション、FAUST DSP記述、Warpマーカーなど、プロユースの機能も揃っている。

**開発状況**

最新リリースはv0.8.3（2024年9月）。直近のコミットは2026年2月とややスローダウン気味だが、プロジェクトは生きている。VST3対応やバグ修正が続いており、実用に耐えるレベルだ。

### 2位：Renoise — トラッカー式キーボード駆動DAW

Renoiseは伝統的なトラッカーUIを備えた商用DAW（EUR 76,00 + VAT）。GUIはあるが、**キーボードだけで全ての操作を完結できる**点でCLI志向のユーザーに刺さる。

**特徴**

- VST2 / VST3 / AU / LADSPA / DSSI 対応
- 数値を直接打ち込むトラッカー方式（パターンシーケンサー）
- LuaスクリプトAPIによる完全自動化
- ヘッドレスレンダリングは公式機能としては限定的だが、Lua経由でバッチ処理を組める
- Linux / macOS / Windows 対応

実際のところ「純粋なCLI」ではないが、ターミナルライクな数値入力とスクリプト制御を重視するなら最良の商用選択肢だ。

### 3位：Tracktion Engine — DAWを自作するC++ライブラリ

Tracktion Engineは単体のDAWアプリではなく、**DAWを自作するためのC++フレームワーク**だ。JUCEモジュールとして提供されており、C++20必須。

**特徴**

- オーディオ編集、MIDIシーケンス、プラグインホストの基本機能をライブラリとして提供
- VST3 / AU対応のプラグインホスト機能込み
- ヘッドレスなCLIアプリケーションを自身でビルド可能
- 商用利用にはライセンス購入が必要

Tracktion Engineは「CLIで動くVST対応アプリをスクラッチで作りたい」という超ニッチなユースケースに答える。とはいえ、そこまでやる人がどれだけいるかは別問題だ。

### 番外編

#### OpenMPT

Windows向けの定番トラッカー。VST対応で、コマンドラインからのレンダリングが可能。軽量で起動が速く、CI/CDにも組み込みやすい。Windows環境に限定されるが、安定感は抜群。

#### Csound

テキストで音響を完全記述する音響プログラミング言語。CsoundVST を介してVSTプラグインとして動作させたり、外部LADSPA/DSSIプラグインをホストすることもできる。ゲームエンジンやライブコーディングとの親和性が高く、「音楽をテキストで書きたい」欲求を最も純粋に満たす。

#### SuperCollider

リアルタイム音響合成サーバー。sc3-pluginsを導入すれば一部VST対応。音楽制作というより音響合成・アルゴリズム作曲の文脈で強力。

## まとめ：自分に合った選択肢はどれ？

| 目的 | おすすめ |
|------|----------|
| PythonでVSTを操作してバッチレンダリングしたい | **DawDreamer** |
| キーボード駆動で高速作曲したい | **Renoise** |
| 音楽をテキストで完全記述したい | **Csound** |
| 自分でDAWを作りたい | **Tracktion Engine** |
| Windowsで軽量トラッカーが欲しい | **OpenMPT** |

筆者の結論としては **DawDreamer** が最もバランスが良い。Pythonエコシステムとの連携（numpy / scipy / librosa）が強力で、Dockerでコンテナ化すればCI/CDパイプラインにも組み込める。CLI好きな音楽プロデューサーにとって、2026年現在もっとも現実的なヘッドレスDAW環境と言えるだろう。
