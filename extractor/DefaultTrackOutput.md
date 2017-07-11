### DefaultTrackOutput

[TrackOutput](../extractor/TrackOutput.md)提取出来的编码数据作为缓冲放在队列中用来消费

![DefaultTrackOutput_structure](../images/DefaultTrackOutput_structure.png)

#### public boolean skipToKeyframeBefore(long timeUs, boolean allowTimeBeyondBuffer)

跳到指定位置之前的关键帧