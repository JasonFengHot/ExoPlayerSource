Implements the internal behavior of ExoPlayerImpl.

####  doSomeWork
````java
  private void doSomeWork() throws ExoPlaybackException, IOException {
    long operationStartTimeMs = SystemClock.elapsedRealtime();
    updatePeriods();
    if (playingPeriodHolder == null) {
      // We're still waiting for the first period to be prepared.
      maybeThrowPeriodPrepareError();
      scheduleNextWork(operationStartTimeMs, PREPARING_SOURCE_INTERVAL_MS);
      return;
    }

    TraceUtil.beginSection("doSomeWork");

    updatePlaybackPositions();
    playingPeriodHolder.mediaPeriod.discardBuffer(playbackInfo.positionUs);

    boolean allRenderersEnded = true;
    boolean allRenderersReadyOrEnded = true;
    for (Renderer renderer : enabledRenderers) {
      // TODO: Each renderer should return the maximum delay before which it wishes to be called
      // again. The minimum of these values should then be used as the delay before the next
      // invocation of this method.
      renderer.render(rendererPositionUs, elapsedRealtimeUs);
      allRenderersEnded = allRenderersEnded && renderer.isEnded();
      // Determine whether the renderer is ready (or ended). If it's not, throw an error that's
      // preventing the renderer from making progress, if such an error exists.
      boolean rendererReadyOrEnded = renderer.isReady() || renderer.isEnded();
      if (!rendererReadyOrEnded) {
        renderer.maybeThrowStreamError();
      }
      allRenderersReadyOrEnded = allRenderersReadyOrEnded && rendererReadyOrEnded;
    }

    if (!allRenderersReadyOrEnded) {
      maybeThrowPeriodPrepareError();
    }

    // The standalone media clock never changes playback parameters, so just check the renderer.
    if (rendererMediaClock != null) {
      PlaybackParameters playbackParameters = rendererMediaClock.getPlaybackParameters();
      if (!playbackParameters.equals(this.playbackParameters)) {
        // TODO: Make LoadControl, period transition position projection, adaptive track selection
        // and potentially any time-related code in renderers take into account the playback speed.
        this.playbackParameters = playbackParameters;
        standaloneMediaClock.synchronize(rendererMediaClock);
        eventHandler.obtainMessage(MSG_PLAYBACK_PARAMETERS_CHANGED, playbackParameters)
            .sendToTarget();
      }
    }

    long playingPeriodDurationUs = timeline.getPeriod(playingPeriodHolder.index, period)
        .getDurationUs();
    if (allRenderersEnded
        && (playingPeriodDurationUs == C.TIME_UNSET
        || playingPeriodDurationUs <= playbackInfo.positionUs)
        && playingPeriodHolder.isLast) {
      setState(ExoPlayer.STATE_ENDED);
      stopRenderers();
    } else if (state == ExoPlayer.STATE_BUFFERING) {
      boolean isNewlyReady = enabledRenderers.length > 0
          ? (allRenderersReadyOrEnded && haveSufficientBuffer(rebuffering))
          : isTimelineReady(playingPeriodDurationUs);
      if (isNewlyReady) {
        setState(ExoPlayer.STATE_READY);
        if (playWhenReady) {
          startRenderers();
        }
      }
    } else if (state == ExoPlayer.STATE_READY) {
      boolean isStillReady = enabledRenderers.length > 0 ? allRenderersReadyOrEnded
          : isTimelineReady(playingPeriodDurationUs);
      if (!isStillReady) {
        rebuffering = playWhenReady;
        setState(ExoPlayer.STATE_BUFFERING);
        stopRenderers();
      }
    }

    if (state == ExoPlayer.STATE_BUFFERING) {
      for (Renderer renderer : enabledRenderers) {
        renderer.maybeThrowStreamError();
      }
    }

    if ((playWhenReady && state == ExoPlayer.STATE_READY) || state == ExoPlayer.STATE_BUFFERING) {
      scheduleNextWork(operationStartTimeMs, RENDERING_INTERVAL_MS);
    } else if (enabledRenderers.length != 0) {
      scheduleNextWork(operationStartTimeMs, IDLE_INTERVAL_MS);
    } else {
      handler.removeMessages(MSG_DO_SOME_WORK);
    }

    TraceUtil.endSection();
  }
````

### updatePeriods
```java
  private void updatePeriods() throws ExoPlaybackException, IOException {
    if (timeline == null) {
      // We're waiting to get information about periods.
      mediaSource.maybeThrowSourceInfoRefreshError();
      return;
    }

    // Update the loading period if required.
    maybeUpdateLoadingPeriod();
    if (loadingPeriodHolder == null || loadingPeriodHolder.isFullyBuffered()) {
      setIsLoading(false);
    } else if (loadingPeriodHolder != null && loadingPeriodHolder.needsContinueLoading) {
      maybeContinueLoading();
    }

    if (playingPeriodHolder == null) {
      // We're waiting for the first period to be prepared.
      return;
    }

    // Update the playing and reading periods.
    while (playingPeriodHolder != readingPeriodHolder
        && rendererPositionUs >= playingPeriodHolder.next.rendererPositionOffsetUs) {
      // All enabled renderers' streams have been read to the end, and the playback position reached
      // the end of the playing period, so advance playback to the next period.
      playingPeriodHolder.release();
      setPlayingPeriodHolder(playingPeriodHolder.next);
      playbackInfo = new PlaybackInfo(playingPeriodHolder.index,
          playingPeriodHolder.startPositionUs);
      updatePlaybackPositions();
      eventHandler.obtainMessage(MSG_POSITION_DISCONTINUITY, playbackInfo).sendToTarget();
    }

    if (readingPeriodHolder.isLast) {
      for (int i = 0; i < renderers.length; i++) {
        Renderer renderer = renderers[i];
        SampleStream sampleStream = readingPeriodHolder.sampleStreams[i];
        // Defer setting the stream as final until the renderer has actually consumed the whole
        // stream in case of playlist changes that cause the stream to be no longer final.
        if (sampleStream != null && renderer.getStream() == sampleStream
            && renderer.hasReadStreamToEnd()) {
          renderer.setCurrentStreamFinal();
        }
      }
      return;
    }

    for (int i = 0; i < renderers.length; i++) {
      Renderer renderer = renderers[i];
      SampleStream sampleStream = readingPeriodHolder.sampleStreams[i];
      if (renderer.getStream() != sampleStream
          || (sampleStream != null && !renderer.hasReadStreamToEnd())) {
        return;
      }
    }

    if (readingPeriodHolder.next != null && readingPeriodHolder.next.prepared) {
      TrackSelectorResult oldTrackSelectorResult = readingPeriodHolder.trackSelectorResult;
      readingPeriodHolder = readingPeriodHolder.next;
      TrackSelectorResult newTrackSelectorResult = readingPeriodHolder.trackSelectorResult;

      boolean initialDiscontinuity =
          readingPeriodHolder.mediaPeriod.readDiscontinuity() != C.TIME_UNSET;
      for (int i = 0; i < renderers.length; i++) {
        Renderer renderer = renderers[i];
        TrackSelection oldSelection = oldTrackSelectorResult.selections.get(i);
        if (oldSelection == null) {
          // The renderer has no current stream and will be enabled when we play the next period.
        } else if (initialDiscontinuity) {
          // The new period starts with a discontinuity, so the renderer will play out all data then
          // be disabled and re-enabled when it starts playing the next period.
          renderer.setCurrentStreamFinal();
        } else if (!renderer.isCurrentStreamFinal()) {
          TrackSelection newSelection = newTrackSelectorResult.selections.get(i);
          RendererConfiguration oldConfig = oldTrackSelectorResult.rendererConfigurations[i];
          RendererConfiguration newConfig = newTrackSelectorResult.rendererConfigurations[i];
          if (newSelection != null && newConfig.equals(oldConfig)) {
            // Replace the renderer's SampleStream so the transition to playing the next period can
            // be seamless.
            Format[] formats = new Format[newSelection.length()];
            for (int j = 0; j < formats.length; j++) {
              formats[j] = newSelection.getFormat(j);
            }
            renderer.replaceStream(formats, readingPeriodHolder.sampleStreams[i],
                readingPeriodHolder.getRendererOffset());
          } else {
            // The renderer will be disabled when transitioning to playing the next period, either
            // because there's no new selection or because a configuration change is required. Mark
            // the SampleStream as final to play out any remaining data.
            renderer.setCurrentStreamFinal();
          }
        }
      }
    }
  }
```