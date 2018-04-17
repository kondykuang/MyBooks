# A2DP SBC 编码流程

## A2DP与音频的接口

        欲流之远者，必浚其泉源。A2DP 必须能读取到 Audio data 数据流（PCM格式），才能谈编码，我们就先来看看A2DP 数据从何而来。

### 1. Audio data 来源

        熟悉Android 架构就会比较了解，Audio data 由 Audio flinger 将上层所有音频源进行合成为一路后输出，在 A2DP 不工作的时候，会直接通过 tinyalsa 接口写入声卡驱动进行发声。当BT A2DP audio设备存在时，音频会优先将 Audio data写入 A2DP，然而 Audio flinger 与 A2DP（or Bluetooth） 运行在不同的进程空间，要进行数据交互，必然用到进程间通信。

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

{% code-tabs %}
{% code-tabs-item title="audio\_a2dp\_hw.cc" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

同样是audio\_hardware下面，企图采用skt\_connect去连接 A2DP\_DATA\_PATH，而注释也说明了意图：connect socket if not yet connected，就连ERROR log也表明此处是想要：opening data socket，看来Audio 与 A2DP 间数据交互可能是采用了socket方式进行进程间通信。

        但是熟悉 socket 编程就会联想到，connect 操作属于 client 端，难道 server 端在 audio 进程？搜索源码对 A2DP\_DATA\_PATH 的引用，发现在：

{% code-tabs %}
{% code-tabs-item title="btif\_a2dp\_control.cc" %}
```cpp
static void btif_a2dp_recv_ctrl_data(void) { 
...
    if (btif_av_stream_ready()) {
        /* Setup audio data channel listener */
        UIPC_Open(*a2dp_uipc, UIPC_CH_ID_AV_AUDIO, btif_a2dp_data_cb,
                  A2DP_DATA_PATH);
    }
    ...
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

此处对 A2DP\_DATA\_PATH 进行Open，看上去像是 server端，打开 UIPC\_Open 的定义：

{% code-tabs %}
{% code-tabs-item title="uipc.cc" %}
```cpp
bool UIPC_Open(tUIPC_STATE& u
ipc, tUIPC_CH_ID ch_id, tUIPC_RCV_CBACK* p_cback,
               const char* socket_path) {
  ...
  uipc_setup_server_locked(uipc, ch_id, socket_path, p_cback);
  return true;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

关键函数 uipc\_setup\_server\_locked证明此处 UIPC\_Open确实是在建立 server socket，可这就奇怪了,明明是要用 socket 进行进程间通信，可为何 client 和 server 端都在 Bluedroid A2DP之中？

        进一步分析不难发现，audio\_a2dp\_hw 这个目录被单独编译成一个动态链接库，而此库应该是运行在 Audio 进程空间，从该目录的 Android.bp 文件可以看出这一点。大概是为了解耦和方便维护，将 原本应该运行在 Audio 端的 socket client放在 Bluedroid 端实现，前面也有提到，实际上 Audio 只关心接口，任何一个实现了 Audio 接口的动态库都可以被加载到 Audio进程空间获取 Audio data，当然前提是此库：audio.xxxx.default.so 中 xxxx 是 Audio 中注册的一个设备吧。

         到此已经很清楚 Audio data的来源了：Bluedroid提供一个动态链接库给Audio，当A2DP工作时，Audio会直接通过此库中 socket client 进行 socket 连接和data 发送，那么 Bluedroid A2DP中同名 socket server 就能收到 Audio data数据了。

 ​       至于 Bluedroid中是在何处处理 data 和 command的，请看上文提到的函数 btif\_a2dp\_recv\_ctrl\_data，从命名可知，此函数即是用于处理 Audio 的命令，Audio data呢，则在 UIPC\_Open的第三个参数 btif\_a2dp\_data\_cb -----回调函数中处理。

### 2. Audio command 来源

 ​       前面以A2DP\_DATA\_PATH为线索分析了 Audio data的来源，，还剩下一个 A2DP\_CTRL\_PATH，以 A2DP\_CTRL\_PATH 为线索就能摸清 command的来源了。

### 3. Audio A2DP 的实现

audio.a2dp.default.so 作为一个 HAL 动态链接库，必须遵循 Android HAL 设计规则。HAL 设计了统一的硬件抽象层接口，任何一个标准的硬件抽象，必须实现此接口。

 HAL的设计初衷是，作为库的使用者不需要知道库的实现细节，统统采用：

* `通过 hw_get_module 加载一个 HAL 库, 并取得 hw_module_t`
* `通过 hw_module_t 中的 hw_module_methods_t 中的 open 方法获取 hw_device_t`
* `hw_device_t 提供顶层接口,并且可以进行扩展`

audio\_a2dp 中定义了一个 hw\_module\_t 结构体：

```cpp
__attribute__((
    visibility("default"))) struct audio_module HAL_MODULE_INFO_SYM = {
    .common =
        {
            .tag = HARDWARE_MODULE_TAG,
            .version_major = 1,
            .version_minor = 0,
            .id = AUDIO_HARDWARE_MODULE_ID,
            .name = "A2DP Audio HW HAL",
            .author = "The Android Open Source Project",
            .methods = &hal_module_methods,
        },
};
```

现在还看不出来 audio\_mudule 是一个 hw\_module\_t，但是找到 audio\_module的定义：

{% code-tabs %}
{% code-tabs-item title="audio.h" %}
```cpp
/**
 * Every hardware module must have a data structure named HAL_MODULE_INFO_SYM
 * and the fields of this data structure must begin with hw_module_t
 * followed by module specific information.
 */
struct audio_module {
    struct hw_module_t common;
};
```
{% endcode-tabs-item %}
{% endcode-tabs %}

audio\_module 等价于一个 hw\_module\_t（请自行查看定义），并对其 hw\_module\_methods\_t 进行了赋值：

```cpp
.methods = &hal_module_methods,
```

来看看 hal\_module\_methods :

{% code-tabs %}
{% code-tabs-item title="audio\_a2dp\_hw.cc" %}
```cpp
static struct hw_module_methods_t hal_module_methods = {
    .open = adev_open,
};
```
{% endcode-tabs-item %}
{% endcode-tabs %}

只实现了 一个open方法，而此open 方法是HAL 强制要求实现的

{% code-tabs %}
{% code-tabs-item title="hardware.h" %}
```cpp
typedef struct hw_module_methods_t {
    /** Open a specific device */
    int (*open)(const struct hw_module_t* module, const char* id,
            struct hw_device_t** device);

} hw_module_methods_t;
```
{% endcode-tabs-item %}
{% endcode-tabs %}

如果想要增加一些接口到methods里面，可以自定义一个 xxx\_module\_methods\_t，将其第一个域定义为 hw\_module\_methods\_t即可，再需要调用自定义方法的时候，将此 hw\_module\_methods\_t 强制转换为 xxx\_module\_methods\_t 即可调用。但是查看 Android AOSP 代码，发现并没有在hw\_module\_methods\_t进行扩展的例子，我想原因是 Android 将 hw\_device\_t 作为顶层抽象，任何扩展添加到 hw\_device\_t 中更合适吧。

从 open 函数指针定义，我们看到其参数是一个hw\_device\_t，HAL 库的使用者（一般是JNI中）拿到此 hw\_device\_t 就可以全权访问该库的所有资源了（当然是通过 hw\_device\_t 暴露出来的资源）。

来看看 aduio\_a2dp 中 device opend 定义：

{% code-tabs %}
{% code-tabs-item title="audio\_a2dp\_hw.cc" %}
```cpp
static int adev_open(const hw_module_t* module, const char* name,
                     hw_device_t** device) {
  struct a2dp_audio_device* adev;
 ​ ...
  adev = (struct a2dp_audio_device*)calloc(1, sizeof(struct a2dp_audio_device));

  if (!adev) return -ENOMEM;

  adev->mutex = new std::recursive_mutex;

  adev->device.common.tag = HARDWARE_DEVICE_TAG;
  adev->device.common.version = AUDIO_DEVICE_API_VERSION_2_0;
  adev->device.common.module = (struct hw_module_t*)module;
  adev->device.common.close = adev_close;

  adev->device.init_check = adev_init_check;
  adev->device.set_voice_volume = adev_set_voice_volume;
  adev->device.set_master_volume = adev_set_master_volume;
  adev->device.set_mode = adev_set_mode;
  adev->device.set_mic_mute = adev_set_mic_mute;
  adev->device.get_mic_mute = adev_get_mic_mute;
  adev->device.set_parameters = adev_set_parameters;
  adev->device.get_parameters = adev_get_parameters;
  adev->device.get_input_buffer_size = adev_get_input_buffer_size;
  adev->device.open_output_stream = adev_open_output_stream;
  adev->device.close_output_stream = adev_close_output_stream;
  adev->device.open_input_stream = adev_open_input_stream;
  adev->device.close_input_stream = adev_close_input_stream;
  adev->device.dump = adev_dump;

  adev->output = NULL;
  *device = &adev->device.common;
  return 0;
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

adev\_open 参数是按照 HAL规约写成，但是其实现中，定义了一个 a2dp\_audio\_device 对象， 并且为它分配了内存，然后进行了很多函数指针赋值。来看看a2dp\_audio\_device定义：

```cpp
struct a2dp_audio_device {
  // Important: device must be first as an audio_hw_device* may be cast to
  // a2dp_audio_device* when the type is implicitly known.
  struct audio_hw_device device;
  std::recursive_mutex* mutex;  // See note below on mutex acquisition order.
  struct a2dp_stream_in* input;
  struct a2dp_stream_out* output;
};
```

a2dp\_audio\_device封装了 audio\_hw\_device ：

{% code-tabs %}
{% code-tabs-item title="audio.h" %}
```cpp
struct audio_hw_device {
    struct hw_device_t common;
    uint32_t (*get_supported_devices)(const struct audio_hw_device *dev);
    int (*init_check)(const struct audio_hw_device *dev);
```
{% endcode-tabs-item %}
{% endcode-tabs %}

其实不难猜到，Audio 模块基于 hw\_device\_t 定义了 audio\_hw\_device，并且将 hw\_device\_t common 设置为其第一个域。然后又扩展了很多自定义函数指针，例如get\_supported\_devices，adev\_set\_voice\_volume等。

注意到在 adev\_open最后：

```cpp
*device = &adev->device.common;
```

将adev-&gt;device.common 赋值给 device，也就是将a2dp\_audio\_device中 hw\_device\_t common 赋值给了函数参数的 hw\_devie\_t device。为什么有这样的操作，来看看 HAL 库是如何使用的就清楚了。以 GPS JNI 为例 简要看看 HAL 库的使用方法：

```cpp
// 1.通过 hw_get_module 获取指定 HAL 库的 hw_module_t
err = hw_get_module(GPS_HARDWARE_MODULE_ID, (hw_module_t const**)&module);
if (err == 0) {
    hw_device_t* device;
    // 2.通过 hw_module_t 中 open 方法获取 hw_device_t
    err = module->methods->open(module, GPS_HARDWARE_MODULE_ID, &device);
    if (err == 0) {
        // 3. 强制转换为自定义的 gps_device_t
        gps_device_t* gps_device = (gps_device_t *)device;
        sGpsInterface = gps_device->get_gps_interface(gps_device);
    }
}
```

通过以上 HAL 库的使用方法印证了：hw\_device\_t 是HAL 设计最上层接口。而且能够通过强制转换获取到自定义的 gps\_device\_t，其原因是在自定义或者说是扩展 hw\_device\_t 的时候都将 hw\_device\_t 类型作为其第一个对象，其实是运用C语言结构体实现了面向对象的继承\(考虑其内存空间，是包含关系\)。

如上简要介绍了 Android HAL 设计思想，现在回到A2DP上，adev\_open 将 audio\_a2dp 中的函数接口绑定到了 audio\_hw\_device 上，由此一来，在Audio 进程中通过 hw\_get\_module 等一系列方法就可以拿到 audio\_a2dp 中的函数j接口了，从而可以向 Bluetooth\_a2dp 进行：

* init\_check
* set\_voice\_volume
* get\_parameters
* open\_output\_stream
* close\_output\_stream

等操作了。现在来看看，Audio data 是如何进入的，在audio\_a2dp扩展的方法中 open\_output\_stream 有可能与写入数据有关（Audio 相对于 A2DP 是写入端）

```cpp
  adev->device.open_output_stream = adev_open_output_stream;

```

open\_output\_stream 指向 本地（audio\_a2dp）adev\_open\_output\_stream函数：

{% code-tabs %}
{% code-tabs-item title="audio\_a2dp\_hw.cc" %}
```cpp
static int adev_open_output_stream(struct audio_hw_device* dev,
                                   UNUSED_ATTR audio_io_handle_t handle,
                                   UNUSED_ATTR audio_devices_t devices,
                                   UNUSED_ATTR audio_output_flags_t flags,
                                   struct audio_config* config,
                                   struct audio_stream_out** stream_out,
                                   UNUSED_ATTR const char* address)

{
  struct a2dp_audio_device* a2dp_dev = (struct a2dp_audio_device*)dev;
  struct a2dp_stream_out* out;
  int ret = 0;

  INFO("opening output");
  // protect against adev->output and stream_out from being inconsistent
  std::lock_guard<std::recursive_mutex> lock(*a2dp_dev->mutex);
  out = (struct a2dp_stream_out*)calloc(1, sizeof(struct a2dp_stream_out));

  if (!out) return -ENOMEM;

  out->stream.common.get_sample_rate = out_get_sample_rate;
  out->stream.common.set_sample_rate = out_set_sample_rate;
  out->stream.common.get_buffer_size = out_get_buffer_size;
  out->stream.common.get_channels = out_get_channels;
  out->stream.common.get_format = out_get_format;
  out->stream.common.set_format = out_set_format;
  out->stream.common.standby = out_standby;
  out->stream.common.dump = out_dump;
  out->stream.common.set_parameters = out_set_parameters;
  out->stream.common.get_parameters = out_get_parameters;
  out->stream.common.add_audio_effect = out_add_audio_effect;
  out->stream.common.remove_audio_effect = out_remove_audio_effect;
  out->stream.get_latency = out_get_latency;
  out->stream.set_volume = out_set_volume;
  out->stream.write = out_write;
  out->stream.get_render_position = out_get_render_position;
  out->stream.get_presentation_position = out_get_presentation_position;
```
{% endcode-tabs-item %}
{% endcode-tabs %}

可见 Audion 将输出抽象为 audio\_stream\_out：

{% code-tabs %}
{% code-tabs-item title="audio.h" %}
```cpp
struct audio_stream_out {
    /**
     * Common methods of the audio stream out.  This *must* be the first member of audio_stream_out
     * as users of this structure will cast a audio_stream to audio_stream_out pointer in contexts
     * where it's known the audio_stream references an audio_stream_out.
     */
    struct audio_stream common;

    /**
     * Write audio buffer to driver. Returns number of bytes written, or a
     * negative status_t. If at least one frame was written successfully prior to the error,
     * it is suggested that the driver return that successful (short) byte count
     * and then return an error in the subsequent call.
     *
     * If set_callback() has previously been called to enable non-blocking mode
     * the write() is not allowed to block. It must write only the number of
     * bytes that currently fit in the driver/hardware buffer and then return
     * this byte count. If this is less than the requested write size the
     * callback function must be called when more space is available in the
     * driver/hardware buffer.
     */
    ssize_t (*write)(struct audio_stream_out *stream, const void* buffer,
                     size_t bytes);

};

```
{% endcode-tabs-item %}
{% endcode-tabs %}

并且在此处分配内存，设置了很多与 audio\_stream\_out 相关的函数指针，例如：

* get\_sample\_rate
* set\_sample\_rate
* set\_parameters

等等，然而 audio\_stream\_out 中最重要的应该是 write 函数指针，应该是用于输出 Audio data的了。write 函数指针指向：

```cpp
 out->stream.write = out_write;
```

来看看 out\_write：

```cpp
static ssize_t out_write(struct audio_stream_out* stream, const void* buffer,
                         size_t bytes) {
  struct a2dp_stream_out* out = (struct a2dp_stream_out*)stream;
  ...
 ​/* only allow autostarting if we are in stopped or standby */
  if ((out->common.state == AUDIO_HA_STATE_STOPPED) ||
      (out->common.state == AUDIO_HA_STATE_STANDBY)) {
    if (start_audio_datapath(&out->common) < 0) {
      goto finish;
    }
  } else if (out->common.state != AUDIO_HA_STATE_STARTED) {
    ERROR("stream not in stopped or standby");
    goto finish;
  }
   
  ...  
  lock.unlock();
  sent = skt_write(out->common.audio_fd, buffer, write_bytes);
  lock.lock();
  ...
  return bytes;
}
```

```cpp
static int skt_write(int fd, const void* p, size_t len) {
 ​   ...
    // do not poll, use blocking send
    OSI_NO_INTR(sent = send(fd, p, len, MSG_NOSIGNAL));
    if (sent == -1) ERROR("write failed with error(%s)", strerror(errno));
 ​   ...
  return (int)count;
}
```

这样就摸清楚了，Audio 是如何向 Bluedroid A2DP 写入 Audio data的了。

然而skt\_write 使用了 audio\_fd 作为文件描述符，然而 adev\_open\_output\_stream 中初始化 audio\_stream\_out 的时候并没有赋值，来看看是 如何赋值的：

```cpp
static int start_audio_datapath(struct ha_stream_common* common) {
 ​ ...
  /* connect socket if not yet connected */
  if (common->audio_fd == AUDIO_SKT_DISCONNECTED) {
    common->audio_fd = skt_connect(HEARING_AID_DATA_PATH, common->buffer_sz);
    if (common->audio_fd < 0) {
      ERROR("Audiopath start failed - error opening data socket");
      goto error;
    }
  }
 ​ ...
  return -1;
}

```

正是在每次 out\_write的时候，都会检查是否 audio\_fd 处于 DISCONNECTED 状态，是则会建立到 socket server端的连接，打通通信通道。



