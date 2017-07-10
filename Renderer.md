![Renderer](/images/Renderer_UML.png)

![Renderer_structure](/images/Renderer_structure.png)

#### int getTrackType()

**TRACK_TYPE_VIDEO** 视频

**TRACK_TYPE_AUDIO** 音频

**TRACK_TYPE_TEXT** 字幕


#### RendererCapabilities getCapabilities()

#### void setIndex(int index)

#### MediaClock getMediaClock()

#### int getState()

#### void enable(RendererConfiguration configuration, Format[] formats, SampleStream stream,long positionUs, boolean joining, long offsetUs) throws ExoPlaybackException

#### void start() throws ExoPlaybackException

#### void replaceStream(Format[] formats, SampleStream stream, long offsetUs) throws ExoPlaybackException

#### SampleStream getStream()

#### boolean hasReadStreamToEnd()

#### void setCurrentStreamFinal()

#### boolean isCurrentStreamFinal()

#### void maybeThrowStreamError() throws IOException

#### void resetPosition(long positionUs) throws ExoPlaybackException

#### void render(long positionUs, long elapsedRealtimeUs) throws ExoPlaybackException

#### boolean isReady()

#### boolean isEnded()

#### void stop() throws ExoPlaybackException

#### void disable()
