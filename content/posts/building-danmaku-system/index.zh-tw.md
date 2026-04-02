---
title: "在 Godot 4 中構建彈幕系統"
date: "2025-05-30T14:32:08-07:00"
categories: [Devlog, Godot4, C#]
tags: [Danmaku, BulletComment, IndieGameDev, HelloWorldGoodbye, Godot4, C#]
description: "使用 Godot 4 和 C# 從零構建一個高效能、可擴展的影片彈幕系統，支援佇列排程、軌道分配與速度修正機制，靈感來自 NicoNico 與 Bilibili 彈幕。"
---

最近我在《Hello, World. Goodbye.》中構建了一個最能表達情緒的功能之一 —— 影片彈幕系統。這是向 NicoNico 式的彈幕留言致敬，也是一種讓玩家在沉默中向虛空吶喊，或是傾聽他人曾經吶喊過的話語的方式。

## 🎯 目標

設計一個模組化、流暢、高效能的彈幕系統，具有以下特點：

- 獨立於主遊戲相機運作（透過專用 Viewport 實現）
- 支援多種內容類型（文字、圖片等）
- 彈幕可均勻分布於各軌道，或採用聚焦式分布風格
- 動態計算彈幕間距，智慧避免重疊

## 🎬 Step-by-Step Construction

### 第一步：從簡單開始

我們從一個基於 `RichTextLabel` 的彈幕節點開始：

![彈幕節點場景結構](./danmaku-scene.png#center)

上圖展示了一個最基礎的彈幕節點場景結構。每條彈幕都是一個 `RichTextLabel` 節點，掛載在彈幕專用的父節點下，便於統一管理和批次操作。

附加的 C# 腳本如下：

```c#
public override void _Process(double delta) {
    Position -= new Vector2(Speed * (float)delta, 0);
    if (Position.X + GetWidth() < 0) QueueFree();
}
```

這個方案一開始運作良好，直到我們發現彈幕出現了奇怪的偏移——即使彈幕移動到 (width, 0)，它似乎還是從不對的位置出現。

![首次嘗試動畫](./first-attempt.gif#center)

彈幕不僅沒有從正確位置出現，還過早地消失了。為什麼會這樣？原來玩家有一個相機，導致螢幕顯示的位置與實際位置並不一致。這就是問題所在。

![玩家相機樹](./player-camera.png#center)

### 第二步：與相機解耦

為了解決這個問題，我們將彈幕系統移動到一個獨立的 Viewport 層（640x360），不再受主相機影響。這樣彈幕無論世界如何移動，都會始終從螢幕右側出現。

![彈幕 Viewport](./danmaku-viewport.png#center)

> 問題：彈幕是如何分布在螢幕上的？

以 B 站演唱會彈幕為例，可以看到螢幕上有多條軌道，彈幕從右向左捲動。

![B站初音演唱會彈幕示例](./danmaku-example.png#center)

實際上，所有彈幕都分布在螢幕上的「隱形軌道」中，如下圖所示：

![彈幕軌道示意](./danmaku-example-track.png#center)

### 第三步：新增軌道管理器

接下來，我們引入了 `DanmakuTrackManager`，用於管理彈幕軌道，確保彈幕不會重疊。

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

系統會根據彈幕節點（如 `RichTextDanmakuLabel`）的尺寸自動計算軌道數量和間距。

### 第四步：新增分配器

為了靈活地管理彈幕分配，我們需要新增一個分配器。因此我建立了一個 `IDanmakuTrackDistributor`，用於控制彈幕分配到哪個軌道。支援兩種分配模式：

- 均勻分配：輪流填充各軌道
- 聚焦分配：集中堆疊到少數軌道，營造視覺衝擊

```c#
public class EvenDistributor : IDanmakuTrackDistributor
{
  public int GetTrackIndex(int danmakuCount, int trackCount)
  {
    return danmakuCount % trackCount;
  }
}
```

這是最簡單的輪詢分配方式。實際應用中，還需結合軌道管理器判斷軌道是否可用。

### 第五步：新增佇列系統

通常我們希望彈幕批次出現但不是一次性全都畫在螢幕上，因此需要一個佇列系統，統一分發彈幕。例如遊戲觸發某事件時，批次新增彈幕。

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

這樣可以在 `_Process` 方法中逐幀檢查佇列和軌道空間，軌道可用時再生成彈幕，避免重疊。

### 第六步：修復超車問題

出現了一個 bug：同一軌道上速度快的彈幕會「超車」慢彈幕，導致重疊。B 站等平台也有類似現象。解決方法是引入「速度修正」機制，確保在軌道新增的彈幕速度不會超過軌道裡已有彈幕的最大速度。

可以在 `DanmakuTrackManager` 中新增如下程式碼：

```c#
// 設置初始速度，限制最大速度防止超車
danmakuData.Speed = Math.Max(GetMaxSpeedInTrack(trackIndex), danmakuData.Speed);
```

然後實現 `GetMaxSpeedInTrack` 方法：

```c#
private float GetMaxSpeedInTrack(int trackIndex)
{
  var track = _tracks[trackIndex];
  if (track.Count == 0) return 0;

  // 遍歷軌道，獲取最大速度
  return track.Max(entry => entry.Entry.Speed);
}
```

### 第七步：系統整合

現在我們已經實現了 `DanmakuTrackManager`、`IDanmakuTrackDistributor`、`DanmakuSystemController` 和 `RichTextDanmakuLabel`，可以在主場景中將它們整合起來。

![彈幕系統結構圖](./system-diag.png#center)

### 最終效果

![最終彈幕效果](./final-result.gif#center)

### 總結

現在的彈幕系統已經模組化、高效能且易於擴展，支援文字、圖片等多種內容類型，彈幕分布可以均勻或聚焦，動態計算間距有效避免重疊。

這也是我用 Godot 4 和 C# 實現複雜功能、同時保持架構清晰的一個實踐。如果你也想在 Godot 4 裡做彈幕系統，希望這篇文章能幫到你！有問題或想法歡迎留言交流。Happy coding!
