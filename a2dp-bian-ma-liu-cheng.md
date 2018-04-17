# A2DP SBC 编码流程

## A2DP与音频的接口

        欲流之远者，必浚其泉源。A2DP 必须能读取到 Audio data 数据流（PCM格式），才能谈编码，我们就先来看看A2DP 数据从何而来。

### 1. Audio data 来源

        熟悉Android 架构就会比较了解，Audio data 由 Audio flinger 将上层所有音频源进行合成为一路后输出，在 BT 不工作的时候，会直接通过 tinyalsa 接口写入声卡驱动进行发声。当BT A2DP audio设备存在时，音频会优先将 Audio data写入 BT，然而 Audio flinger 与 BT 运行在不同的进程空间，要进行数据交互，必然用到进程间通信。

        打开 Bluedroid 代码目录，第一个文件夹的名称 audio\_a2dp\_hw 第一个字母就暗示了其与 Audio 必定有关联：

```text
<audio_a2dp_hw>   #dir
    <include>     #dir
        audio_a2dp_hw.h
    <src>         #dir
        audio_a2dp_hw.cc
        audio_a2dp_hw_utils.cc
```

而 hw 一词正是从 audio 的视角，将 aduio 输出设备抽象为 audio output hardware（例如speaker、earphone、wired headset、bluetooth headset 等）。 抽象的目的是为了面向接口编程，而不管其具体实现，所以 A2DP 对于 Audio 来说可能与其他 Audio HW 也没有两样。

        有抽象，必然有接口，现在我们就来找找 Audio抽象接口， 打开头文件 audio\_a2dp\_hw.h，发现 Bluedroid给 audio\_a2dp\_hw取了一个名字 “audio.a2dp”：

```cpp
#define A2DP_AUDIO_HARDWARE_INTERFACE "audio.a2dp"
#define A2DP_CTRL_PATH "/data/misc/bluedroid/.a2dp_ctrl"
#define A2DP_DATA_PATH "/data/misc/bluedroid/.a2dp_data"
```

并且还定义了两个 PATH，查看 A2DP\_DATA\_PATH 的引用，发现：

```cpp
static int start_audio_datapath(struct a2dp_stream_common* common) {
  ...

  /* connect socket if not yet connected */
  if (common->audio_fd == AUDIO_SKT_DISCONNECTED) {
    common->audio_fd = skt_connect(A2DP_DATA_PATH, common->buffer_sz);
    if (common->audio_fd < 0) {
      ERROR("Audiopath start failed - error opening data socket");
      goto error;
    }
  }
    ...
}
```

同样是audio\_hardware下面，企图采用skt\_connect去连接 A2DP\_DATA\_PATH，而注释也说明了意图：connect socket if not yet connected，就连ERROR log也表明此处是想要：opening data socket，看来Audio 与 A2DP 间数据交互可能是采用了socket方式进行进程间通信

### 2. Audio command 来源

## Getting Super Powers

Becoming a super hero is a fairly straight forward process:

```
$ give me super-powers
```

{% hint style="info" %}
 Super-powers are granted randomly so please submit an issue if you're not happy with yours.
{% endhint %}

Once you're strong enough, save the world:

```
// Ain't no code for that yet, sorry
echo 'You got to trust me on this, I saved the world'
```



