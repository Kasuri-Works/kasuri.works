---
title: "Godot 4 で作るビデオ弾幕システム"
date: "2025-05-30T14:32:08-07:00"
lang: ja
categories: [Devlog, Godot4, C#]
tags: [Danmaku, BulletComment, IndieGameDev, HelloWorldGoodbye, Godot4, C#]
description: "Godot 4 と C# を使って、ニコニコ風の弾幕コメントシステムをゼロから構築したプロセスを紹介します。独立した Viewport、トラック制御、速度補正などを備えた高性能な実装例です。"
---

最近、**Hello, World. Goodbye.** で特に感情表現が豊かな機能として実装しているのが、*弾幕コメントシステム*です。これはニコニコ動画の弾幕コメントへのオマージュであり、プレイヤーが静かに叫んだり、他の人の声を聞いたりできる仕組みです。

## 🎯 目標

モジュール化され、滑らかで高性能な弾幕（弾幕コメント）システムを設計します。要件は以下の通りです：

- メインゲームカメラと独立して動作（専用 Viewport を使用）
- 複数のコンテンツタイプ（テキスト、画像）に対応
- 弾幕をトラックごとに均等または集中して分配
- 間隔を動的に計算し、重なりを回避

## 🎬 実装ステップ

### ステップ 1: シンプルな実装から開始

まずは `RichTextLabel` ベースの弾幕ノードを作成し、右から左へ移動させます。

![Damaku Scene](./danmaku-scene.png#center)

C# コード例：

```c#
public override void _Process(double delta) {
  Position -= new Vector2(Speed * (float)delta, 0);
  if (Position.X + GetWidth() < 0) QueueFree();
}
```

この実装では、弾幕が意図しない位置から出現し、早すぎるタイミングで消えてしまう問題が発生しました。

![First Attempt Animation](./first-attempt.gif#center)

原因はプレイヤーにカメラがアタッチされており、画面上の座標と実際の位置が一致しないためです。

![Player Camera Tree](./player-camera.png#center)

### ステップ 2: カメラからの分離

この問題を解決するため、弾幕システムをメインカメラの影響を受けない専用の Viewport（例：640x360）に移動しました。これで、ワールドの動きに関係なく、常に画面右端から弾幕が流れます。

![Danmaku Viewport](./danmaku-viewport.png#center)

> Q: 弾幕はどのように画面上に分配される？

Bilibili のコンサート動画例を見ると、複数のトラックに弾幕が流れているのが分かります。

![Example Miku Concert Danmaku - Bilibili](./danmaku-example.png#center)

弾幕は画面上の見えないトラックに分配されているイメージです。

![Danmaku Tracks](./danmaku-example-track.png#center)

### ステップ 3: トラックマネージャの導入

`DanmakuTrackManager` を導入し、各トラック（行）ごとに弾幕が重ならないよう管理します。

```c#
public class DanmakuTrackManager
{
  private readonly List<List<DanmakuTrackEntry>> _tracks;
  private readonly Node _danmakuLayer;
  private readonly int _trackCount;
  private readonly float _trackHeight;
  private readonly float _screenWidth;
  private readonly float _spacing;

  public bool IsTrackAvailable(int trackIndex)
  {
    var track = _tracks[trackIndex];
    if (track.Count == 0) return true;

    var last = track[^1];
    float lastRight = last.Entry.Position.X + last.Entry.Call("GetWidth").AsSingle();
    return lastRight + 20f < _screenWidth;
  }
}
```

トラック数や間隔は、サンプル弾幕（例：`RichTextDanmakuLabel`）のサイズから自動計算します。

### ステップ 4: 配分ロジックの追加

`IDanmakuTrackDistributor` を実装し、弾幕のトラック割り当て方法を制御します。

均等分配（ラウンドロビン）と集中分配（少数トラックに集中）を実装：

```c#
public class EvenDistributor : IDanmakuTrackDistributor
{
  public int GetTrackIndex(int danmakuCount, int trackCount)
  {
    return danmakuCount % trackCount;
  }
}
```

実際には、トラックが空いているかどうかをトラックマネージャで確認する必要があります。

### ステップ 5: キューシステムの導入

大量の弾幕を一度に追加する場合、キューに溜めてから順次分配します。

```c#
public partial class DanmakuSystemController : Node, ISystemModule
{
  private Queue<DanmakuQueueEntry> _danmakuQueue;
  private DanmakuTrackManager _trackManager;
  private double _lastSpawnTime = 0;

  public override void _Ready()
  {
    _danmakuQueue = new Queue<DanmakuQueueEntry>();
    _trackManager = new DanmakuTrackManager();
  }

  public void AddDanmaku(DanmakuQueueEntry entry)
  {
    _danmakuQueue.Enqueue(entry);
    ProcessQueue();
  }

  public override void _Process(double delta)
  {
    if (_danmakuQueue.Count == 0) return;
    // Peek queue and if time passed, spawn a danmaku
    if (Time.GetTicksMsec() > _lastSpawnTime + SpawnInterval)
    {
      // If the next track is available on the track manager
      if (_trackManager.HasTrackSpace())
      {
        var danmakuData = _danmakuQueue.Dequeue();
        _trackManager.AddDanmakuToTrack(danmakuData.Data);
        _lastSpawnTime = Time.GetTicksMsec();
      }
    }
  }
}
```

このコードで、弾幕をキューに溜め、トラックに空きがあれば順次追加します。

### ステップ 6: オーバーラン問題の修正

速い弾幕が遅い弾幕を追い越して重なるバグが発生しました。これを防ぐため、「スピードクリップ」を導入し、前の弾幕が安全領域を抜けるまで次の弾幕を追加しません。

```c#
// Set initial speed, cap the speed to prevent overrun
danmakuData.Speed = Math.Max(GetMaxSpeedInTrack(trackIndex), danmakuData.Speed);
```

`GetMaxSpeedInTrack` メソッド例：

```c#
private float GetMaxSpeedInTrack(int trackIndex)
{
  var track = _tracks[trackIndex];
  if (track.Count == 0) return 0;

  // Loop through track to get max speed
  return track.Max(entry => entry.Entry.Speed);
}
```

### ステップ 7: 全体の統合

`DanmakuTrackManager`、`IDanmakuTrackDistributor`、`DanmakuSystemController`、`RichTextDanmakuLabel` を組み合わせて、メインシーンに統合します。

![Danmaku System Diagram](./system-diag.png#center)

### 最終結果

![Final Danmaku Result](./final-result.gif#center)

### まとめ

この弾幕システムは、モジュールごとに役割を分けて設計しているため、機能追加や調整がしやすくなっています。テキストや画像など複数のコンテンツタイプに対応し、弾幕の分配や重なり回避も柔軟に制御できます。Godot 4 と C# でこうした仕組みを作る際の参考になればうれしいです。ご質問やフィードバックがあればお気軽にどうぞ。
