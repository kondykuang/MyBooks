# SBC 编码流程

## Bluedroid与音频的接口

欲流之远者，必浚其泉源。Bluedroid 必须能读取到音频的 PCM 数据流，才能谈编码，然后在音频与 Bluedroid 的交互中，除了`Audio data`还有`Audio command` ，例如`A2DP_CTRL_CMD_START`，`A2DP_CTRL_CMD_STOP` 等，我们就先来看看Bluedroid 数据与命令从何而来。

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



