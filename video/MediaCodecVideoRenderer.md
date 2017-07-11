![video_crop](../images/video_crop.png)

crop-left | 裁剪左侧
crop-right | 裁剪右侧
crop-bottom | 裁剪下侧
crop-top | 裁剪上侧

#### STANDARD_LONG_EDGE_VIDEO_PX

标准视频像素宽度

**1920, 1600, 1440, 1280, 960, 854, 640, 540, 480**


#### protected boolean processOutputBuffer(long positionUs, long elapsedRealtimeUs, MediaCodec codec, ByteBuffer buffer, int bufferIndex, int bufferFlags, long bufferPresentationTimeUs,boolean shouldSkip)

![processOutputBuffer](../images/MediaCodecVideoRenderer_processOutputBuffer.png)

```
    if (shouldSkip) {
      skipOutputBuffer(codec, bufferIndex);
      return true;
    }
```





