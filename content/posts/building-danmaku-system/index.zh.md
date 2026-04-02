---
title: "在 Godot 4 中构建视频弹幕系统"
date: "2025-05-30T14:32:08-07:00"
categories: [Devlog, Godot4, C#]
tags: [Danmaku, BulletComment, IndieGameDev, HelloWorldGoodbye, Godot4, C#]
description: "使用 Godot 4 和 C# 从零构建一个高性能、可扩展的视频弹幕系统，支持队列调度、轨道分配与速度修正机制，灵感来自 NicoNico 与 Bilibili 弹幕。"
---

最近我在《Hello, World. Goodbye.》中构建了一个最能表达情绪的功能之一 —— 视频弹幕系统。这是向 NicoNico 式的弹幕评论致敬，也是一种让玩家在沉默中向虚空呐喊，或是倾听他人曾经呐喊过的话语的方式。

## 🎯 目标

设计一个模块化、流畅、高性能的弹幕系统，具有以下特点：

- 独立于主游戏相机运行（通过专用 Viewport 实现）
- 支持多种内容类型（文本、图片等）
- 弹幕可均匀分布于各轨道，或采用聚焦式分布风格
- 动态计算弹幕间距，智能避免重叠

## 🎬 Step-by-Step Construction

### 第一步：从简单开始

我们从一个基于 `RichTextLabel` 的弹幕节点开始：

![弹幕节点场景结构](./danmaku-scene.png#center)

上图展示了一个最基础的弹幕节点场景结构。每条弹幕都是一个 `RichTextLabel` 节点，挂载在弹幕专用的父节点下，便于统一管理和批量操作。

附加的 C# 脚本如下：

```c#
public override void _Process(double delta) {
    Position -= new Vector2(Speed * (float)delta, 0);
    if (Position.X + GetWidth() < 0) QueueFree();
}
```

这个方案一开始运行良好，直到我们发现弹幕出现了奇怪的偏移——即使弹幕移动到 (width, 0)，它似乎还是从不对的位置出现。

![首次尝试动画](./first-attempt.gif#center)

弹幕不仅没有从正确位置出现，还过早地消失了。为什么会这样？原来玩家有一个摄像机，导致屏幕显示的位置与实际位置并不一致。这就是问题所在。

![玩家摄像机树](./player-camera.png#center)

### 第二步：与摄像机解耦

为了解决这个问题，我们将弹幕系统移动到一个独立的 Viewport 层（640x360），不再受主摄像机影响。这样弹幕无论世界如何移动，都会始终从屏幕右侧出现。

![弹幕 Viewport](./danmaku-viewport.png#center)

> 问题：弹幕是如何分布在屏幕上的？

以 B 站演唱会弹幕为例，可以看到屏幕上有多条轨道，弹幕从右向左滚动。

![B站初音演唱会弹幕示例](./danmaku-example.png#center)

实际上，所有弹幕都分布在屏幕上的“隐形轨道”中，如下图所示：

![弹幕轨道示意](./danmaku-example-track.png#center)

### 第三步：添加轨道管理器

接下来，我们引入了 `DanmakuTrackManager`，用于管理弹幕轨道，确保弹幕不会重叠。

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

系统会根据弹幕节点（如 `RichTextDanmakuLabel`）的尺寸自动计算轨道数量和间距。

### 第四步：添加分配器

为了灵活性的管理弹幕分配，我们需要添加一个分配器。因此我创建了一个 `IDanmakuTrackDistributor`，用于控制弹幕分配到哪个轨道。支持两种分配模式：

- 均匀分配：轮流填充各轨道
- 聚焦分配：集中堆叠到少数轨道，营造视觉冲击

```c#
public class EvenDistributor : IDanmakuTrackDistributor
{
  public int GetTrackIndex(int danmakuCount, int trackCount)
  {
    return danmakuCount % trackCount;
  }
}
```

这是最简单的轮询分配方式。实际应用中，还需结合轨道管理器判断轨道是否可用。

### 第五步：添加队列系统

通常我们希望弹幕批量出现但是不是一次性全都画在屏幕上，因此需要一个队列系统，统一分发弹幕。例如游戏触发某事件时，批量添加弹幕。

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

这样可以在 `_Process` 方法中逐帧检查队列和轨道空间，轨道可用时再生成弹幕，避免重叠。

### 第六步：修复超车问题

出现了一个 bug：同一轨道上速度快的弹幕会“超车”慢弹幕，导致重叠。B 站等平台也有类似现象。解决方法是引入“速度修正”机制，确保在轨道添加的新的弹幕速度不会超过轨道里已有弹幕的最大速度。

可以在在 `DanmakuTrackManager` 中添加如下代码：

```c#
// 设置初始速度，限制最大速度防止超车
danmakuData.Speed = Math.Max(GetMaxSpeedInTrack(trackIndex), danmakuData.Speed);
```

然后实现 `GetMaxSpeedInTrack` 方法：

```c#
private float GetMaxSpeedInTrack(int trackIndex)
{
  var track = _tracks[trackIndex];
  if (track.Count == 0) return 0;

  // 遍历轨道，获取最大速度
  return track.Max(entry => entry.Entry.Speed);
}
```

### 第七步：系统整合

现在我们已经实现了 `DanmakuTrackManager`、`IDanmakuTrackDistributor`、`DanmakuSystemController` 和 `RichTextDanmakuLabel`，可以在主场景中将它们整合起来。

![弹幕系统结构图](./system-diag.png#center)

### 最终效果

![最终弹幕效果](./final-result.gif#center)

### 总结

现在的弹幕系统已经模块化、高性能且易于扩展，支持文本、图片等多种内容类型，弹幕分布可以均匀或聚焦，动态计算间距有效避免重叠。

这也是我用 Godot 4 和 C# 实现复杂功能、同时保持架构清晰的一个实践。如果你也想在 Godot 4 里做弹幕系统，希望这篇文章能帮到你！有问题或想法欢迎留言交流。Happy coding!
