---
title : Android TTSManager - Android Focus Ducking 추가
categories : [Kotlin, tips]
tags : [tts, texttospeech, android, kotlin, queue, watchdog]
img_path: /assets/img/
pin : true
---

TTS Manager를 만들어서 사용하다보니 샵앤뮤직이나 다른 뮤직 앱이랑 소리가 겹쳐져서
TTS 음성 가독성이 떨어지는 일이 발생.

일단 샵앤뮤직에 Ducking 소스가 추가되어있는지 테스트해봤더니 뮤직앱답게 추가되어 있음.
TTS Manager에 Android Focus Ducking 추가

---

## 추가된 부분


```kotlin
// ducking start!
private fun requestAudioFocus() {
    if(IS_USE_AUDIOFOCUS != 1) return
    val attributes = AudioAttributes.Builder()
        .setUsage(AudioAttributes.USAGE_ASSISTANCE_ACCESSIBILITY)
        .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH)
        .build()

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        audioFocusRequest = AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK)
            .setAudioAttributes(attributes)
            .setOnAudioFocusChangeListener(focusChangeListener)
            .build()

        audioManager.requestAudioFocus(audioFocusRequest!!)
    } else {
        audioManager.requestAudioFocus(
            focusChangeListener,
            AudioManager.STREAM_MUSIC,
            AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK
        )
    }
}

// ducking end!
private fun abandonAudioFocus() {
    if(IS_USE_AUDIOFOCUS != 1) return

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        audioFocusRequest?.let {
            audioManager.abandonAudioFocusRequest(it)
        }
        audioFocusRequest = null
    } else {
        audioManager.abandonAudioFocus(focusChangeListener)
    }
}
```

---

## 전체소스

```kotlin
class TTSManager(
    context: Context
) : TextToSpeech.OnInitListener {

    companion object {
        private const val TAG = "TTSManager"
        private const val WATCHDOG_TIMEOUT_MS = 30_000L
        private const val RECOVER_WINDOW_MS = 120_000L
        private const val MAX_RECOVER_IN_WINDOW = 3
        // Audio Focus Ducking 사용 안하려면 0, 사용하려면 1
        private const val IS_USE_AUDIOFOCUS = 1
    }

    private val appContext = context.applicationContext
    private var tts: TextToSpeech = TextToSpeech(appContext, this)

    private val audioManager = appContext.getSystemService(Context.AUDIO_SERVICE) as AudioManager

    private var audioFocusRequest: AudioFocusRequest? = null
    private val focusChangeListener = AudioManager.OnAudioFocusChangeListener { }

    private val queue: LinkedList<String> = LinkedList()
    private var isInitialized = false
    private var isSpeakingNow = false
    private var isShutdown = false
    private var currentSpeakingText: String? = null
    private var currentUtteranceId: String? = null
    private val mainHandler = Handler(Looper.getMainLooper())
    private var generation = 0
    private val recoverTimestamps: LinkedList<Long> = LinkedList()

    private val watchdogRunnable = Runnable {
        if (isShutdown) return@Runnable
        if (isSpeakingNow) {
            recoverTts("timeout")
        }
    }

    override fun onInit(status: Int) {
        if (isShutdown) return

        if (status == TextToSpeech.SUCCESS) {
            try {
                val result = tts.setLanguage(Locale.KOREAN)

                if (result == TextToSpeech.LANG_MISSING_DATA || result == TextToSpeech.LANG_NOT_SUPPORTED) {
                    isInitialized = false
                    return
                }

                val audioAttributes = AudioAttributes.Builder()
                    .setUsage(AudioAttributes.USAGE_ASSISTANCE_ACCESSIBILITY)
                    .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH)
                    .build()

                tts.setAudioAttributes(audioAttributes)

                bindUtteranceListener()

                isInitialized = true
                playNext()

            } catch (e: Exception) {
                isInitialized = false
            }
        } else {
            isInitialized = false
        }
    }

    fun speak(text: String?) {
        if (isShutdown) return

        val safeText = text?.trim().orEmpty()
        if (safeText.isEmpty()) return

        queue.add(safeText)
        playNext()
    }

    private fun playNext() {
        if (isShutdown) return
        if (!isInitialized) return
        if (isSpeakingNow) return

        val nextText = queue.poll() ?: return

        currentSpeakingText = nextText
        currentUtteranceId = System.currentTimeMillis().toString()

        val utteranceId = currentUtteranceId!!

        val result = try {
            val params = Bundle().apply {
                putFloat(TextToSpeech.Engine.KEY_PARAM_VOLUME, 1.0f)
                putFloat(TextToSpeech.Engine.KEY_PARAM_PAN, 0.0f)
            }

            requestAudioFocus()

            tts.speak(
                nextText,
                TextToSpeech.QUEUE_FLUSH,
                params,
                utteranceId
            )
        } catch (e: Exception) {
            TextToSpeech.ERROR
        }

        if (result == TextToSpeech.ERROR) {
            isSpeakingNow = false
            abandonAudioFocus()
            recoverTts("speak_error")
            return
        }

        startWatchdog()
    }

    // ducking start!
    private fun requestAudioFocus() {
        if(IS_USE_AUDIOFOCUS != 1) return
        val attributes = AudioAttributes.Builder()
            .setUsage(AudioAttributes.USAGE_ASSISTANCE_ACCESSIBILITY)
            .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH)
            .build()

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            audioFocusRequest = AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK)
                .setAudioAttributes(attributes)
                .setOnAudioFocusChangeListener(focusChangeListener)
                .build()

            audioManager.requestAudioFocus(audioFocusRequest!!)
        } else {
            audioManager.requestAudioFocus(
                focusChangeListener,
                AudioManager.STREAM_MUSIC,
                AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK
            )
        }
    }

    // ducking end!
    private fun abandonAudioFocus() {
        if(IS_USE_AUDIOFOCUS != 1) return

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            audioFocusRequest?.let {
                audioManager.abandonAudioFocusRequest(it)
            }
            audioFocusRequest = null
        } else {
            audioManager.abandonAudioFocus(focusChangeListener)
        }
    }

    private fun bindUtteranceListener() {
        val myGeneration = ++generation

        tts.setOnUtteranceProgressListener(object : UtteranceProgressListener() {

            override fun onStart(utteranceId: String?) {
                if (isShutdown) return
                if (myGeneration != generation) return

                Log.e("jhbae", "onStart: $utteranceId")

                isSpeakingNow = true
                startWatchdog()
            }

            override fun onDone(utteranceId: String?) {
                if (isShutdown) return
                if (myGeneration != generation) return

                Log.e("jhbae", "onDone: $utteranceId")
                handleUtteranceFinished(utteranceId)
            }

            override fun onError(utteranceId: String?) {
                if (isShutdown) return
                if (myGeneration != generation) return

                Log.e("jhbae", "onError: $utteranceId")
                handleUtteranceError(utteranceId, "error_legacy")
            }

            override fun onError(utteranceId: String?, errorCode: Int) {
                if (isShutdown) return
                if (myGeneration != generation) return

                Log.e("jhbae", "onError: $utteranceId, code=$errorCode")
                handleUtteranceError(utteranceId, "error_code=$errorCode")
            }

            override fun onStop(utteranceId: String?, interrupted: Boolean) {
                if (isShutdown) return
                if (myGeneration != generation) return

                Log.e("jhbae", "onStop: $utteranceId, interrupted=$interrupted")
            }
        })
    }

    private fun handleUtteranceFinished(utteranceId: String?) {
        if (utteranceId != null && currentUtteranceId != null && utteranceId != currentUtteranceId) {
            Log.e("jhbae", "handleUtteranceFinished onDone utteranceId 값: $utteranceId (current=$currentUtteranceId)")
            return
        }

        stopWatchdog()

        isSpeakingNow = false
        currentSpeakingText = null
        currentUtteranceId = null

        abandonAudioFocus()

        playNext()
    }

    private fun handleUtteranceError(utteranceId: String?, reason: String) {
        if (utteranceId != null && currentUtteranceId != null && utteranceId != currentUtteranceId) {
            Log.e("jhbae", "handleUtteranceError error utteranceId 값: $utteranceId (current=$currentUtteranceId)")
            return
        }

        stopWatchdog()
        isSpeakingNow = false

        abandonAudioFocus()

        recoverTts(reason)
    }

    private fun startWatchdog() {
        stopWatchdog()
        mainHandler.postDelayed(watchdogRunnable, WATCHDOG_TIMEOUT_MS)
    }

    private fun stopWatchdog() {
        mainHandler.removeCallbacks(watchdogRunnable)
    }

    private fun recoverTts(reason: String) {
        if (isShutdown) return

        if (!canRecoverNow()) {
            Log.e("jhbae", "reason=$reason")
            stopInternal(resetCurrent = false)
            return
        }

        Log.e("jhbae", "reason=$reason")
        stopWatchdog()

        val retryText = currentSpeakingText

        isInitialized = false
        isSpeakingNow = false
        currentSpeakingText = null
        currentUtteranceId = null

        if (!retryText.isNullOrBlank()) {
            queue.addFirst(retryText)
        }

        try {
            tts.stop()
        } catch (e: Exception) {
            Log.e("jhbae", "recoverTts- tts.stop() 에러!!", e)
        }

        try {
            tts.shutdown()
        } catch (e: Exception) {
            Log.e("jhbae", "recoverTts- tts.shutdown() 에러!!", e)
        }

        abandonAudioFocus()

        tts = TextToSpeech(appContext, this)
    }

    private fun canRecoverNow(): Boolean {
        val now = System.currentTimeMillis()

        while (recoverTimestamps.isNotEmpty() && now - recoverTimestamps.first > RECOVER_WINDOW_MS) {
            recoverTimestamps.removeFirst()
        }

        recoverTimestamps.add(now)

        return recoverTimestamps.size <= MAX_RECOVER_IN_WINDOW
    }

    fun stop() {
        if (isShutdown) return
        stopInternal(resetCurrent = true)
    }

    fun clear() {
        if (isShutdown) return
        queue.clear()
        stopInternal(resetCurrent = true)
    }

    private fun stopInternal(resetCurrent: Boolean) {
        stopWatchdog()

        try {
            tts.stop()
        } catch (e: Exception) {
            Log.e("jhbae", "stopInternal-tts.stop() 에러!!", e)
        }

        isSpeakingNow = false

        abandonAudioFocus()

        if (resetCurrent) {
            currentSpeakingText = null
            currentUtteranceId = null
        }
    }

    fun shutdown() {
        if (isShutdown) return

        isShutdown = true
        stopWatchdog()
        queue.clear()

        isInitialized = false
        isSpeakingNow = false
        currentSpeakingText = null
        currentUtteranceId = null

        try {
            tts.stop()
        } catch (e: Exception) {
            Log.e("jhbae", "tts.stop() 에러!!", e)
        }

        try {
            tts.shutdown()
        } catch (e: Exception) {
            Log.e("jhbae", "tts.shutdown() 에러!!", e)
        }

        abandonAudioFocus()
    }
}
```

