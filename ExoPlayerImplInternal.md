Implements the internal behavior of ExoPlayerImpl.

####  doSomeWork()

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
