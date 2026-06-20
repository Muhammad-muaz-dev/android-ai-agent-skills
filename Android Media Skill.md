Scope
This skill governs every media surface in an Android app — acquisition, processing, playback, storage, and permissions. It begins where a media feature is requested and ends when the feature is lifecycle-safe, resource-clean, and production-ready.

It does not own UI business logic, navigation, ViewModel implementation, networking, or database architecture.

Quick Decision Tree
Need a media feature?
│
├─ Record audio                     → MediaRecorder (§A-1) or AudioRecord for raw PCM (§A-2)
├─ Play audio / video               → Media3 / ExoPlayer (§B)
├─ Record video                     → CameraX VideoCapture (§C-2) or MediaRecorder video (§C-3)
├─ Capture photos                   → CameraX ImageCapture (§C-1)
├─ Display camera preview           → CameraX Preview (§C)
├─ Change pitch / speed / effects   → Voice processing pipeline (§D)
├─ Reverse audio                    → Reverse PCM pipeline (§D-4)
├─ Speech-to-text                   → SpeechRecognizer (§E)
├─ Text-to-speech                   → TextToSpeech (§F)
├─ Visualize audio amplitude        → Waveform canvas view (§G)
├─ Trim / compress / merge media    → Background MediaProcessor (§H)
├─ Save to gallery / MediaStore     → Media storage strategy (§I)
├─ Share a media file               → FileProvider (§I-3)
│
└─ Need permissions first?          → Permission handler (§J) — always before any media op

Mandatory Rules
#	Rule
R1	Never perform media I/O, encoding, decoding, or processing on the main thread. Use Dispatchers.IO or a dedicated HandlerThread.
R2	Every media component (MediaRecorder, ExoPlayer, CameraX, SpeechRecognizer, TextToSpeech, AudioTrack) must be released in the matching lifecycle callback — no exceptions.
R3	Always check and request runtime permissions before accessing the microphone, camera, or storage. Handle denial gracefully with a fallback state.
R4	Use Media3 / ExoPlayer for all audio/video playback. Do not use MediaPlayer in new code.
R5	Use CameraX for all camera work. Do not use Camera2 directly unless CameraX cannot satisfy the requirement.
R6	Use Scoped Storage / MediaStore for saving to shared storage. Never hardcode file paths. Never use Environment.getExternalStorageDirectory() on API 29+.
R7	Share files through FileProvider only. Never expose raw file:// URIs to other apps on API 24+.
R8	Handle audio focus for all playback. Request focus before playing; release when done or interrupted.
R9	Handle interruptions (incoming calls, other apps stealing focus) by pausing/ducking and resuming correctly.
R10	Expose media state as a sealed class / StateFlow to the calling layer. Never surface raw platform exceptions.
R11	All media operations must be lifecycle-aware — bind to LifecycleOwner or cancel in onStop/onDestroy.
R12	Validate file format, MIME type, and size before processing. Reject invalid or corrupted input with a typed error state.
Implementation Checklist
Verify every item before marking a media feature complete.

Before starting
 Permissions declared in AndroidManifest.xml
 Runtime permission flow implemented (§J)
 FileProvider configured in manifest for file sharing
Audio / Video recording
 MediaRecorder / AudioRecord initialized with explicit format, encoder, and output path
 Recording started/stopped on background thread
 pause() / resume() handled for API 24+
 release() called in onStop or onDestroy
 Interruption (audio focus loss) pauses recording
Playback
 ExoPlayer built with DefaultMediaItem and MediaSource
 Audio focus requested via AudioFocusRequest
 Player released in onStop (background playback uses MediaSessionService)
 Playback state exposed via StateFlow<PlaybackState>
 Error listener converts PlaybackException to typed state
Camera
 ProcessCameraProvider bound to LifecycleOwner
 Camera use cases unbound before rebinding
 Orientation handled via OrientationEventListener
 Camera resources released by CameraX lifecycle binding (no manual release needed)
Media storage
 Files saved via MediaStore.Images/Audio/Video.Media.EXTERNAL_CONTENT_URI
 ContentValues includes DISPLAY_NAME, MIME_TYPE, RELATIVE_PATH
 IS_PENDING flag used during write; cleared on success
 Temp files cleaned up in finally blocks
State & errors
 Sealed MediaState class covers all states (Idle, Recording, Playing, Paused, Error, Completed)
 All exceptions caught and mapped to MediaState.Error
 Retry mechanism offered for recoverable errors
Dependency Reference
// app/build.gradle.kts
android {
    buildFeatures { viewBinding = true }
}
dependencies {
    // Media3 / ExoPlayer (playback)
    implementation("androidx.media3:media3-exoplayer:1.4.1")
    implementation("androidx.media3:media3-ui:1.4.1")
    implementation("androidx.media3:media3-session:1.4.1")   // background playback
    // CameraX (camera, photo, video capture)
    val cameraxVersion = "1.3.4"
    implementation("androidx.camera:camera-core:$cameraxVersion")
    implementation("androidx.camera:camera-camera2:$cameraxVersion")
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
    implementation("androidx.camera:camera-video:$cameraxVersion")
    implementation("androidx.camera:camera-view:$cameraxVersion")
    implementation("androidx.camera:camera-extensions:$cameraxVersion")
    // Coroutines (background processing)
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
    // Lifecycle (lifecycle-aware components)
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.6")
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.6")
}

<!-- AndroidManifest.xml — declare all permissions your feature uses -->
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />        <!-- API 33+ -->
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />        <!-- API 33+ -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />       <!-- API 33+ -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />                                              <!-- API ≤ 32 -->
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
    android:maxSdkVersion="28" />                                              <!-- API ≤ 28 only -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<!-- FileProvider for sharing files -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_provider_paths" />
</provider>

<!-- res/xml/file_provider_paths.xml -->
<paths>
    <cache-path name="media_cache" path="media/" />
    <external-cache-path name="external_media_cache" path="media/" />
</paths>

§A — Audio Recording
§A-1 — MediaRecorder (compressed output: AAC, MP3, OGG)
Use for standard voice memos, voice messages, and any feature that needs a compressed audio file.

// media/recorder/AudioRecorder.kt
class AudioRecorder(private val context: Context) {
    sealed class RecorderState {
        data object Idle       : RecorderState()
        data object Recording  : RecorderState()
        data object Paused     : RecorderState()
        data class  Error(val message: String, val cause: Throwable? = null) : RecorderState()
        data class  Completed(val outputFile: File) : RecorderState()
    }
    private val _state = MutableStateFlow<RecorderState>(RecorderState.Idle)
    val state: StateFlow<RecorderState> = _state.asStateFlow()
    private var recorder: MediaRecorder? = null
    private var outputFile: File? = null
    fun start(outputDir: File = context.cacheDir) {
        if (_state.value !is RecorderState.Idle) return
        try {
            val file = File(outputDir, "rec_${System.currentTimeMillis()}.m4a")
                .also { outputFile = it }
            recorder = (if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S)
                MediaRecorder(context) else @Suppress("DEPRECATION") MediaRecorder()
            ).apply {
                setAudioSource(MediaRecorder.AudioSource.MIC)
                setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
                setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
                setAudioSamplingRate(44_100)
                setAudioEncodingBitRate(128_000)
                setAudioChannels(1)
                setOutputFile(file.absolutePath)
                // Noise suppression, echo cancellation, AGC — applied at AudioSource level on
                // devices that support it; no explicit API needed for MIC source.
                prepare()
                start()
            }
            _state.value = RecorderState.Recording
        } catch (e: Exception) {
            _state.value = RecorderState.Error("Failed to start recording", e)
            release()
        }
    }
    fun pause() {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.N) return
        if (_state.value !is RecorderState.Recording) return
        try {
            recorder?.pause()
            _state.value = RecorderState.Paused
        } catch (e: IllegalStateException) {
            _state.value = RecorderState.Error("Failed to pause", e)
        }
    }
    fun resume() {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.N) return
        if (_state.value !is RecorderState.Paused) return
        try {
            recorder?.resume()
            _state.value = RecorderState.Recording
        } catch (e: IllegalStateException) {
            _state.value = RecorderState.Error("Failed to resume", e)
        }
    }
    fun stop() {
        try {
            recorder?.stop()
            outputFile?.let { _state.value = RecorderState.Completed(it) }
        } catch (e: RuntimeException) {
            outputFile?.delete() // incomplete file
            _state.value = RecorderState.Error("Recording failed", e)
        } finally {
            release()
        }
    }
    /** Current amplitude — poll at 50–100 ms intervals for waveform visualization. */
    fun getMaxAmplitude(): Int = recorder?.maxAmplitude ?: 0
    fun release() {
        recorder?.runCatching { stop() }
        recorder?.release()
        recorder = null
        if (_state.value is RecorderState.Recording || _state.value is RecorderState.Paused) {
            _state.value = RecorderState.Idle
        }
    }
}
// Fragment / ViewModel usage
class RecorderViewModel(application: Application) : AndroidViewModel(application) {
    private val recorder = AudioRecorder(application)
    val recorderState = recorder.state
    fun startRecording() = recorder.start()
    fun pauseRecording() = recorder.pause()
    fun resumeRecording() = recorder.resume()
    fun stopRecording()  = recorder.stop()
    override fun onCleared() {
        super.onCleared()
        recorder.release()
    }
}

§A-2 — AudioRecord (raw PCM — for waveform, real-time processing)
Use when you need raw PCM samples for FFT, waveform rendering, or voice processing.

class RawAudioRecorder(private val scope: CoroutineScope) {
    private val sampleRate   = 44_100
    private val channelConfig = AudioFormat.CHANNEL_IN_MONO
    private val audioFormat  = AudioFormat.ENCODING_PCM_16BIT
    private val bufferSize   = AudioRecord.getMinBufferSize(sampleRate, channelConfig, audioFormat)
    private val _samples = MutableSharedFlow<ShortArray>(extraBufferCapacity = 64)
    val samples: SharedFlow<ShortArray> = _samples.asSharedFlow()
    private var audioRecord: AudioRecord? = null
    private var recordingJob: Job? = null
    @SuppressLint("MissingPermission") // Caller must check RECORD_AUDIO first
    fun start() {
        audioRecord = AudioRecord(
            MediaRecorder.AudioSource.MIC,
            sampleRate, channelConfig, audioFormat, bufferSize
        ).also { it.startRecording() }
        recordingJob = scope.launch(Dispatchers.IO) {
            val buffer = ShortArray(bufferSize / 2)
            while (isActive) {
                val read = audioRecord?.read(buffer, 0, buffer.size) ?: break
                if (read > 0) _samples.emit(buffer.copyOf(read))
            }
        }
    }
    fun stop() {
        recordingJob?.cancel()
        audioRecord?.stop()
        audioRecord?.release()
        audioRecord = null
    }
}

§B — Audio & Video Playback (Media3 / ExoPlayer)
§B-1 — Foreground (activity/fragment-bound) playback
// media/player/MediaPlayerManager.kt
class MediaPlayerManager(
    private val context: Context,
    private val lifecycleOwner: LifecycleOwner,
    private val scope: CoroutineScope,
) {
    sealed class PlaybackState {
        data object Idle       : PlaybackState()
        data object Buffering  : PlaybackState()
        data object Playing    : PlaybackState()
        data object Paused     : PlaybackState()
        data object Ended      : PlaybackState()
        data class  Error(val message: String, val code: Int) : PlaybackState()
    }
    private val _playbackState = MutableStateFlow<PlaybackState>(PlaybackState.Idle)
    val playbackState: StateFlow<PlaybackState> = _playbackState.asStateFlow()
    private val _progress = MutableStateFlow(0L)
    val progressMs: StateFlow<Long> = _progress.asStateFlow()
    private var progressJob: Job? = null
    val player: ExoPlayer = ExoPlayer.Builder(context)
        .setAudioAttributes(
            AudioAttributes.Builder()
                .setUsage(C.USAGE_MEDIA)
                .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
                .build(),
            /* handleAudioFocus = */ true
        )
        .setHandleAudioBecomingNoisy(true) // auto-pause on headphone unplug
        .build()
        .also { exo ->
            exo.addListener(object : Player.Listener {
                override fun onPlaybackStateChanged(state: Int) {
                    _playbackState.value = when (state) {
                        Player.STATE_BUFFERING -> PlaybackState.Buffering
                        Player.STATE_READY     ->
                            if (exo.playWhenReady) PlaybackState.Playing else PlaybackState.Paused
                        Player.STATE_ENDED     -> PlaybackState.Ended
                        else                   -> PlaybackState.Idle
                    }
                }
                override fun onIsPlayingChanged(isPlaying: Boolean) {
                    if (_playbackState.value !in listOf(PlaybackState.Buffering, PlaybackState.Ended)) {
                        _playbackState.value =
                            if (isPlaying) PlaybackState.Playing else PlaybackState.Paused
                    }
                    if (isPlaying) startProgressTracking() else progressJob?.cancel()
                }
                override fun onPlayerError(error: PlaybackException) {
                    _playbackState.value = PlaybackState.Error(
                        error.message ?: "Playback error",
                        error.errorCode
                    )
                }
            })
            // Tie player lifecycle to the provided LifecycleOwner
            lifecycleOwner.lifecycle.addObserver(object : DefaultLifecycleObserver {
                override fun onStop(owner: LifecycleOwner)    { exo.pause() }
                override fun onDestroy(owner: LifecycleOwner) { release() }
            })
        }
    fun play(uri: Uri) {
        player.setMediaItem(MediaItem.fromUri(uri))
        player.prepare()
        player.playWhenReady = true
    }
    fun playUrl(url: String) = play(Uri.parse(url))
    fun playFile(file: File) = play(Uri.fromFile(file))
    fun setPlaylist(uris: List<Uri>) {
        player.setMediaItems(uris.map { MediaItem.fromUri(it) })
        player.prepare()
        player.playWhenReady = true
    }
    fun pause()  = player.pause()
    fun resume() = player.play()
    fun seekTo(positionMs: Long) = player.seekTo(positionMs)
    fun setSpeed(speed: Float)   = player.setPlaybackSpeed(speed)
    val durationMs: Long get()   = player.duration.coerceAtLeast(0L)
    private fun startProgressTracking() {
        progressJob?.cancel()
        progressJob = scope.launch {
            while (isActive) {
                _progress.value = player.currentPosition
                delay(200)
            }
        }
    }
    fun release() {
        progressJob?.cancel()
        player.release()
    }
}
// Fragment usage
class PlayerFragment : Fragment() {
    private lateinit var playerManager: MediaPlayerManager
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        playerManager = MediaPlayerManager(
            context      = requireContext(),
            lifecycleOwner = viewLifecycleOwner,
            scope        = viewLifecycleOwner.lifecycleScope,
        )
        // Attach to PlayerView (Media3 UI)
        binding.playerView.player = playerManager.player
        playerManager.playUrl("https://example.com/audio.mp3")
        viewLifecycleOwner.lifecycleScope.launch {
            playerManager.playbackState.collect { state ->
                binding.playButton.isEnabled = state !is MediaPlayerManager.PlaybackState.Buffering
                when (state) {
                    is MediaPlayerManager.PlaybackState.Error ->
                        showError(state.message)
                    else -> Unit
                }
            }
        }
    }
}

§B-2 — Background playback (MediaSessionService)
// media/player/PlaybackService.kt
@AndroidEntryPoint
class PlaybackService : MediaSessionService() {
    private lateinit var player: ExoPlayer
    private lateinit var mediaSession: MediaSession
    override fun onCreate() {
        super.onCreate()
        player = ExoPlayer.Builder(this)
            .setAudioAttributes(
                AudioAttributes.Builder()
                    .setUsage(C.USAGE_MEDIA)
                    .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
                    .build(),
                true
            )
            .setHandleAudioBecomingNoisy(true)
            .build()
        mediaSession = MediaSession.Builder(this, player).build()
    }
    override fun onGetSession(controllerInfo: MediaSession.ControllerInfo) = mediaSession
    override fun onDestroy() {
        mediaSession.release()
        player.release()
        super.onDestroy()
    }
}
// Declare in AndroidManifest.xml:
// <service android:name=".media.player.PlaybackService"
//     android:foregroundServiceType="mediaPlayback"
//     android:exported="true">
//   <intent-filter>
//     <action android:name="androidx.media3.session.MediaSessionService"/>
//   </intent-filter>
// </service>

§C — Camera (CameraX)
§C-1 — Photo capture
// media/camera/CameraManager.kt
class CameraManager(
    private val context: Context,
    private val lifecycleOwner: LifecycleOwner,
    private val previewView: PreviewView,
) {
    sealed class CameraState {
        data object Idle      : CameraState()
        data object Ready     : CameraState()
        data object Capturing : CameraState()
        data class  PhotoSaved(val uri: Uri)   : CameraState()
        data class  Error(val message: String) : CameraState()
    }
    private val _state = MutableStateFlow<CameraState>(CameraState.Idle)
    val state: StateFlow<CameraState> = _state.asStateFlow()
    private var imageCapture: ImageCapture? = null
    private var lensFacing = CameraSelector.LENS_FACING_BACK
    suspend fun startCamera() {
        val provider = ProcessCameraProvider.getInstance(context).await()
        val preview = Preview.Builder().build()
            .also { it.setSurfaceProvider(previewView.surfaceProvider) }
        imageCapture = ImageCapture.Builder()
            .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
            .build()
        val selector = CameraSelector.Builder()
            .requireLensFacing(lensFacing)
            .build()
        try {
            provider.unbindAll()
            provider.bindToLifecycle(lifecycleOwner, selector, preview, imageCapture)
            _state.value = CameraState.Ready
        } catch (e: Exception) {
            _state.value = CameraState.Error("Camera bind failed: ${e.message}")
        }
    }
    fun capturePhoto(outputDir: File = context.cacheDir) {
        val capture = imageCapture ?: return
        _state.value = CameraState.Capturing
        val file = File(outputDir, "photo_${System.currentTimeMillis()}.jpg")
        val outputOptions = ImageCapture.OutputFileOptions.Builder(file).build()
        capture.takePicture(
            outputOptions,
            ContextCompat.getMainExecutor(context),
            object : ImageCapture.OnImageSavedCallback {
                override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                    _state.value = CameraState.PhotoSaved(output.savedUri ?: Uri.fromFile(file))
                }
                override fun onError(e: ImageCaptureException) {
                    _state.value = CameraState.Error("Capture failed: ${e.message}")
                }
            }
        )
    }
    fun flipCamera() {
        lensFacing = if (lensFacing == CameraSelector.LENS_FACING_BACK)
            CameraSelector.LENS_FACING_FRONT else CameraSelector.LENS_FACING_BACK
        // Re-bind to apply flip
    }
}

§C-2 — Video recording (CameraX VideoCapture)
private var videoCapture: VideoCapture<Recorder>? = null
private var activeRecording: Recording? = null
fun startVideoCapture(lifecycleOwner: LifecycleOwner, provider: ProcessCameraProvider) {
    val recorder = Recorder.Builder()
        .setQualitySelector(QualitySelector.from(Quality.HIGHEST))
        .build()
    videoCapture = VideoCapture.withOutput(recorder)
    val selector = CameraSelector.DEFAULT_BACK_CAMERA
    val preview  = Preview.Builder().build()
        .also { it.setSurfaceProvider(previewView.surfaceProvider) }
    provider.unbindAll()
    provider.bindToLifecycle(lifecycleOwner, selector, preview, videoCapture)
}
@SuppressLint("MissingPermission")
fun startRecording(outputFile: File) {
    val fileOutput = FileOutputOptions.Builder(outputFile).build()
    activeRecording = videoCapture?.output
        ?.prepareRecording(context, fileOutput)
        ?.withAudioEnabled()
        ?.start(ContextCompat.getMainExecutor(context)) { event ->
            when (event) {
                is VideoRecordEvent.Start    -> _state.value = CameraState.Recording
                is VideoRecordEvent.Finalize ->
                    if (event.hasError()) _state.value = CameraState.Error("Video error: ${event.error}")
                    else _state.value = CameraState.VideoSaved(event.outputResults.outputUri)
            }
        }
}
fun stopRecording() = activeRecording?.stop().also { activeRecording = null }

§C-3 — Orientation handling
// Attach in Fragment.onViewCreated
val orientationListener = object : OrientationEventListener(requireContext()) {
    override fun onOrientationChanged(orientation: Int) {
        val rotation = when (orientation) {
            in 45..134  -> Surface.ROTATION_270
            in 135..224 -> Surface.ROTATION_180
            in 225..314 -> Surface.ROTATION_90
            else        -> Surface.ROTATION_0
        }
        imageCapture?.targetRotation = rotation
        videoCapture?.targetRotation = rotation
    }
}
orientationListener.enable()
viewLifecycleOwner.lifecycle.addObserver(object : DefaultLifecycleObserver {
    override fun onStop(owner: LifecycleOwner) = orientationListener.disable()
    override fun onStart(owner: LifecycleOwner) = orientationListener.enable()
})

§D — Voice Processing Pipeline
All processing runs on Dispatchers.IO. The pipeline is a chain of transforms applied to a PCM ShortArray.

§D-1 — Pipeline architecture
// media/voice/VoiceProcessor.kt
/** A single transform step in the voice pipeline. */
fun interface AudioTransform {
    fun process(samples: ShortArray, sampleRate: Int): ShortArray
}
class VoiceProcessor {
    private val transforms = mutableListOf<AudioTransform>()
    fun addTransform(t: AudioTransform)    = apply { transforms.add(t) }
    fun clearTransforms()                   = transforms.clear()
    suspend fun process(input: ShortArray, sampleRate: Int): ShortArray =
        withContext(Dispatchers.Default) {
            transforms.fold(input) { acc, t -> t.process(acc, sampleRate) }
        }
}

§D-2 — Pitch shift (semitone-based)
class PitchShiftTransform(private val semitones: Float) : AudioTransform {
    override fun process(samples: ShortArray, sampleRate: Int): ShortArray {
        val factor = 2f.pow(semitones / 12f)
        val outputSize = (samples.size / factor).toInt()
        val output = ShortArray(outputSize)
        for (i in output.indices) {
            val srcIdx = (i * factor).toInt().coerceIn(0, samples.size - 1)
            output[i] = samples[srcIdx]
        }
        return output
    }
}
// Presets — compose with VoiceProcessor
val chipmunk  = PitchShiftTransform(+8f)
val deepVoice = PitchShiftTransform(-6f)
val robot     = PitchShiftTransform(0f) // combine with echo for robot effect

§D-3 — Speed change
class SpeedChangeTransform(private val speed: Float) : AudioTransform {
    override fun process(samples: ShortArray, sampleRate: Int): ShortArray {
        val outputSize = (samples.size / speed).toInt()
        val output = ShortArray(outputSize)
        for (i in output.indices) {
            val srcIdx = (i * speed).toInt().coerceIn(0, samples.size - 1)
            output[i] = samples[srcIdx]
        }
        return output
    }
}

§D-4 — Reverse audio
class ReverseTransform : AudioTransform {
    override fun process(samples: ShortArray, sampleRate: Int): ShortArray =
        ShortArray(samples.size) { i -> samples[samples.size - 1 - i] }
}

§D-5 — Echo
class EchoTransform(
    private val delayMs: Int = 300,
    private val decay: Float = 0.5f,
) : AudioTransform {
    override fun process(samples: ShortArray, sampleRate: Int): ShortArray {
        val delaySamples = (sampleRate * delayMs / 1000)
        val output = samples.copyOf()
        for (i in delaySamples until output.size) {
            output[i] = (output[i] + output[i - delaySamples] * decay)
                .toInt().coerceIn(Short.MIN_VALUE.toInt(), Short.MAX_VALUE.toInt()).toShort()
        }
        return output
    }
}

§D-6 — Write processed PCM to file
suspend fun writePcmToWav(samples: ShortArray, sampleRate: Int, outputFile: File) =
    withContext(Dispatchers.IO) {
        val byteCount = samples.size * 2
        DataOutputStream(outputFile.outputStream().buffered()).use { out ->
            // WAV header
            out.writeBytes("RIFF")
            out.writeInt(Integer.reverseBytes(36 + byteCount))
            out.writeBytes("WAVEfmt ")
            out.writeInt(Integer.reverseBytes(16))
            out.writeShort(java.lang.Short.reverseBytes(1).toInt())   // PCM
            out.writeShort(java.lang.Short.reverseBytes(1).toInt())   // mono
            out.writeInt(Integer.reverseBytes(sampleRate))
            out.writeInt(Integer.reverseBytes(sampleRate * 2))
            out.writeShort(java.lang.Short.reverseBytes(2).toInt())   // block align
            out.writeShort(java.lang.Short.reverseBytes(16).toInt())  // bits per sample
            out.writeBytes("data")
            out.writeInt(Integer.reverseBytes(byteCount))
            for (sample in samples) {
                out.writeShort(java.lang.Short.reverseBytes(sample).toInt())
            }
        }
    }

§E — Speech Recognition
// media/speech/SpeechRecognitionManager.kt
class SpeechRecognitionManager(
    private val context: Context,
    private val scope: CoroutineScope,
) {
    sealed class RecognitionState {
        data object Idle        : RecognitionState()
        data object Listening   : RecognitionState()
        data class  Partial(val text: String)  : RecognitionState()
        data class  Result(val text: String)   : RecognitionState()
        data class  Error(val code: Int, val message: String) : RecognitionState()
    }
    private val _state = MutableStateFlow<RecognitionState>(RecognitionState.Idle)
    val state: StateFlow<RecognitionState> = _state.asStateFlow()
    private var recognizer: SpeechRecognizer? = null
    fun start(languageTag: String = Locale.getDefault().toLanguageTag()) {
        if (!SpeechRecognizer.isRecognitionAvailable(context)) {
            _state.value = RecognitionState.Error(-1, "Speech recognition not available on this device")
            return
        }
        recognizer = SpeechRecognizer.createSpeechRecognizer(context).apply {
            setRecognitionListener(object : RecognitionListener {
                override fun onReadyForSpeech(p: Bundle?)      { _state.value = RecognitionState.Listening }
                override fun onBeginningOfSpeech()             {}
                override fun onRmsChanged(rmsdB: Float)        {}
                override fun onBufferReceived(buffer: ByteArray?) {}
                override fun onEndOfSpeech()                   {}
                override fun onPartialResults(partial: Bundle?) {
                    partial?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)
                        ?.firstOrNull()
                        ?.let { _state.value = RecognitionState.Partial(it) }
                }
                override fun onResults(results: Bundle?) {
                    results?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)
                        ?.firstOrNull()
                        ?.let { _state.value = RecognitionState.Result(it) }
                        ?: run { _state.value = RecognitionState.Error(-1, "No results") }
                }
                override fun onError(errorCode: Int) {
                    _state.value = RecognitionState.Error(
                        errorCode, speechErrorMessage(errorCode)
                    )
                }
                override fun onEvent(eventType: Int, params: Bundle?) {}
            })
        }
        val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
            putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
            putExtra(RecognizerIntent.EXTRA_LANGUAGE, languageTag)
            putExtra(RecognizerIntent.EXTRA_PARTIAL_RESULTS, true)
            putExtra(RecognizerIntent.EXTRA_MAX_RESULTS, 1)
        }
        recognizer?.startListening(intent)
    }
    fun stop() {
        recognizer?.stopListening()
        _state.value = RecognitionState.Idle
    }
    fun release() {
        recognizer?.destroy()
        recognizer = null
        _state.value = RecognitionState.Idle
    }
    private fun speechErrorMessage(code: Int) = when (code) {
        SpeechRecognizer.ERROR_AUDIO                 -> "Audio recording error"
        SpeechRecognizer.ERROR_CLIENT                -> "Client error"
        SpeechRecognizer.ERROR_INSUFFICIENT_PERMISSIONS -> "Missing RECORD_AUDIO permission"
        SpeechRecognizer.ERROR_NETWORK               -> "Network error"
        SpeechRecognizer.ERROR_NETWORK_TIMEOUT       -> "Network timeout"
        SpeechRecognizer.ERROR_NO_MATCH              -> "No speech matched"
        SpeechRecognizer.ERROR_RECOGNIZER_BUSY       -> "Recognizer busy"
        SpeechRecognizer.ERROR_SERVER                -> "Server error"
        SpeechRecognizer.ERROR_SPEECH_TIMEOUT        -> "No speech detected"
        else                                         -> "Unknown error ($code)"
    }
}

§F — Text-to-Speech
// media/tts/TtsManager.kt
class TtsManager(private val context: Context) {
    sealed class TtsState {
        data object Uninitialized : TtsState()
        data object Ready         : TtsState()
        data object Speaking      : TtsState()
        data class  Error(val message: String) : TtsState()
        data class  UnsupportedLanguage(val locale: Locale) : TtsState()
    }
    private val _state = MutableStateFlow<TtsState>(TtsState.Uninitialized)
    val state: StateFlow<TtsState> = _state.asStateFlow()
    private var tts: TextToSpeech? = null
    fun initialize(locale: Locale = Locale.getDefault()) {
        tts = TextToSpeech(context) { status ->
            if (status == TextToSpeech.SUCCESS) {
                val result = tts?.setLanguage(locale)
                _state.value = when {
                    result == LANG_MISSING_DATA || result == LANG_NOT_SUPPORTED ->
                        TtsState.UnsupportedLanguage(locale)
                    else -> TtsState.Ready
                }
                tts?.setOnUtteranceProgressListener(object : UtteranceProgressListener() {
                    override fun onStart(utteranceId: String?)    { _state.value = TtsState.Speaking }
                    override fun onDone(utteranceId: String?)     { _state.value = TtsState.Ready }
                    override fun onError(utteranceId: String?)    { _state.value = TtsState.Error("TTS error") }
                })
            } else {
                _state.value = TtsState.Error("TTS initialization failed")
            }
        }
    }
    fun speak(text: String, rate: Float = 1.0f, pitch: Float = 1.0f) {
        if (_state.value !is TtsState.Ready) return
        tts?.setSpeechRate(rate)
        tts?.setPitch(pitch)
        tts?.speak(text, TextToSpeech.QUEUE_FLUSH, null, "utterance_${System.currentTimeMillis()}")
    }
    fun stop()    = tts?.stop()
    fun release() {
        tts?.stop()
        tts?.shutdown()
        tts = null
        _state.value = TtsState.Uninitialized
    }
    // Tie to lifecycle in Fragment
    fun bindToLifecycle(owner: LifecycleOwner) {
        owner.lifecycle.addObserver(object : DefaultLifecycleObserver {
            override fun onCreate(owner: LifecycleOwner)  = initialize()
            override fun onDestroy(owner: LifecycleOwner) = release()
        })
    }
}

§G — Audio Waveform Visualization
// media/waveform/WaveformView.kt
class WaveformView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0,
) : View(context, attrs, defStyleAttr) {
    private val barPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = context.getColor(R.color.waveform_bar)
        strokeCap = Paint.Cap.ROUND
    }
    private val progressPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = context.getColor(R.color.waveform_progress)
        strokeCap = Paint.Cap.ROUND
    }
    private val amplitudes = mutableListOf<Float>()
    private var playbackProgress = 0f  // 0.0 – 1.0
    private val barWidth  = 6f.dp(context)
    private val barGap    = 3f.dp(context)
    /** Call at 50–100 ms intervals during recording. Value 0–32767. */
    fun addAmplitude(amplitude: Int) {
        amplitudes.add(amplitude / 32767f)
        if (amplitudes.size > maxBars()) amplitudes.removeAt(0)
        invalidate()
    }
    /** Set from playback position — value 0.0 to 1.0. */
    fun setProgress(progress: Float) {
        playbackProgress = progress.coerceIn(0f, 1f)
        invalidate()
    }
    fun setAmplitudes(data: List<Float>) {
        amplitudes.clear()
        amplitudes.addAll(data)
        invalidate()
    }
    private fun maxBars() = ((width / (barWidth + barGap)).toInt()).coerceAtLeast(1)
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        if (amplitudes.isEmpty()) return
        val centerY  = height / 2f
        val progressX = width * playbackProgress
        amplitudes.forEachIndexed { index, amp ->
            val x = index * (barWidth + barGap) + barWidth / 2f
            val barHeight = (amp * height * 0.8f).coerceAtLeast(4f)
            val paint = if (x <= progressX) progressPaint else barPaint
            paint.strokeWidth = barWidth
            canvas.drawLine(x, centerY - barHeight / 2f, x, centerY + barHeight / 2f, paint)
        }
    }
    private fun Float.dp(ctx: Context) = this * ctx.resources.displayMetrics.density
}

§H — Media Processing (Trim, Compress, Merge)
All heavy operations run on Dispatchers.IO. Results surface as sealed states.

// media/processor/MediaProcessor.kt
object MediaProcessor {
    sealed class ProcessResult {
        data class  Success(val outputFile: File) : ProcessResult()
        data class  Error(val message: String, val cause: Throwable? = null) : ProcessResult()
    }
    /** Trim an audio/video file using MediaMuxer + MediaExtractor. */
    suspend fun trim(
        inputFile: File,
        outputFile: File,
        startMs: Long,
        endMs: Long,
    ): ProcessResult = withContext(Dispatchers.IO) {
        runCatching {
            val extractor = MediaExtractor().apply { setDataSource(inputFile.absolutePath) }
            val muxer = MediaMuxer(outputFile.absolutePath, MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4)
            val trackMap = mutableMapOf<Int, Int>()
            for (i in 0 until extractor.trackCount) {
                val format = extractor.getTrackFormat(i)
                trackMap[i] = muxer.addTrack(format)
            }
            muxer.start()
            val buffer = ByteBuffer.allocate(1024 * 1024)
            val info   = MediaCodec.BufferInfo()
            trackMap.forEach { (extractorTrack, muxerTrack) ->
                extractor.selectTrack(extractorTrack)
                extractor.seekTo(startMs * 1000L, MediaExtractor.SEEK_TO_CLOSEST_SYNC)
                while (true) {
                    val size = extractor.readSampleData(buffer, 0)
                    if (size < 0 || extractor.sampleTime > endMs * 1000L) break
                    info.offset         = 0
                    info.size           = size
                    info.presentationTimeUs = extractor.sampleTime - startMs * 1000L
                    info.flags          = extractor.sampleFlags
                    muxer.writeSampleData(muxerTrack, buffer, info)
                    extractor.advance()
                }
                extractor.unselectTrack(extractorTrack)
            }
            muxer.stop(); muxer.release(); extractor.release()
            ProcessResult.Success(outputFile)
        }.getOrElse { e ->
            outputFile.delete()
            ProcessResult.Error("Trim failed: ${e.message}", e)
        }
    }
    /** Generate a thumbnail from a video file at the given time. */
    suspend fun videoThumbnail(videoFile: File, timeMs: Long = 0L): Bitmap? =
        withContext(Dispatchers.IO) {
            runCatching {
                val retriever = MediaMetadataRetriever().apply {
                    setDataSource(videoFile.absolutePath)
                }
                val bmp = retriever.getFrameAtTime(timeMs * 1000L, MediaMetadataRetriever.OPTION_CLOSEST_SYNC)
                retriever.release()
                bmp
            }.getOrNull()
        }
    /** Extract basic metadata from any media file. */
    suspend fun extractMetadata(file: File): Map<String, String?> =
        withContext(Dispatchers.IO) {
            val r = MediaMetadataRetriever().apply { setDataSource(file.absolutePath) }
            mapOf(
                "duration"  to r.extractMetadata(MediaMetadataRetriever.METADATA_KEY_DURATION),
                "mimeType"  to r.extractMetadata(MediaMetadataRetriever.METADATA_KEY_MIMETYPE),
                "bitrate"   to r.extractMetadata(MediaMetadataRetriever.METADATA_KEY_BITRATE),
                "width"     to r.extractMetadata(MediaMetadataRetriever.METADATA_KEY_VIDEO_WIDTH),
                "height"    to r.extractMetadata(MediaMetadataRetriever.METADATA_KEY_VIDEO_HEIGHT),
                "artist"    to r.extractMetadata(MediaMetadataRetriever.METADATA_KEY_ARTIST),
                "title"     to r.extractMetadata(MediaMetadataRetriever.METADATA_KEY_TITLE),
            ).also { r.release() }
        }
}

§I — Media Storage
§I-1 — Save to MediaStore (shared gallery / Music)
// media/storage/MediaStorage.kt
object MediaStorage {
    /** Save an image file to the shared Photos/Pictures collection. */
    suspend fun saveImage(context: Context, sourceFile: File, displayName: String): Uri? =
        withContext(Dispatchers.IO) {
            val values = ContentValues().apply {
                put(MediaStore.Images.Media.DISPLAY_NAME, displayName)
                put(MediaStore.Images.Media.MIME_TYPE,    "image/jpeg")
                put(MediaStore.Images.Media.RELATIVE_PATH, Environment.DIRECTORY_PICTURES + "/MyApp")
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q)
                    put(MediaStore.Images.Media.IS_PENDING, 1)
            }
            val resolver = context.contentResolver
            val uri = resolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values) ?: return@withContext null
            try {
                resolver.openOutputStream(uri)?.use { out ->
                    sourceFile.inputStream().use { it.copyTo(out) }
                }
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                    values.clear()
                    values.put(MediaStore.Images.Media.IS_PENDING, 0)
                    resolver.update(uri, values, null, null)
                }
                uri
            } catch (e: Exception) {
                resolver.delete(uri, null, null)
                null
            }
        }
    /** Save an audio file to the shared Music collection. */
    suspend fun saveAudio(context: Context, sourceFile: File, displayName: String): Uri? =
        withContext(Dispatchers.IO) {
            val values = ContentValues().apply {
                put(MediaStore.Audio.Media.DISPLAY_NAME, displayName)
                put(MediaStore.Audio.Media.MIME_TYPE,    "audio/mp4")
                put(MediaStore.Audio.Media.RELATIVE_PATH, Environment.DIRECTORY_MUSIC + "/MyApp")
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q)
                    put(MediaStore.Audio.Media.IS_PENDING, 1)
            }
            val resolver = context.contentResolver
            val uri = resolver.insert(MediaStore.Audio.Media.EXTERNAL_CONTENT_URI, values) ?: return@withContext null
            try {
                resolver.openOutputStream(uri)?.use { out ->
                    sourceFile.inputStream().use { it.copyTo(out) }
                }
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                    values.clear()
                    values.put(MediaStore.Audio.Media.IS_PENDING, 0)
                    resolver.update(uri, values, null, null)
                }
                uri
            } catch (e: Exception) {
                resolver.delete(uri, null, null)
                null
            }
        }
    /** Delete a temporary cache file safely. */
    fun deleteCacheFile(file: File) {
        if (file.exists() && file.canonicalPath.startsWith(file.parentFile?.canonicalPath ?: "")) {
            file.delete()
        }
    }
}

§I-2 — App-private cache directory
// For temporary processing files — no permissions needed
fun tempMediaFile(context: Context, extension: String): File {
    val dir = File(context.cacheDir, "media").also { it.mkdirs() }
    return File(dir, "tmp_${System.currentTimeMillis()}.$extension")
}

§I-3 — Share via FileProvider
fun shareMediaFile(context: Context, file: File, mimeType: String) {
    val uri = FileProvider.getUriForFile(
        context,
        "${context.packageName}.fileprovider",
        file
    )
    val intent = Intent(Intent.ACTION_SEND).apply {
        type = mimeType
        putExtra(Intent.EXTRA_STREAM, uri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    context.startActivity(Intent.createChooser(intent, "Share media"))
}

§J — Permission Handling
§J-1 — Required permissions by feature
Feature	API < 29	API 29–32	API 33+
Record audio	RECORD_AUDIO	RECORD_AUDIO	RECORD_AUDIO
Camera	CAMERA	CAMERA	CAMERA
Read images	READ_EXTERNAL_STORAGE	READ_EXTERNAL_STORAGE	READ_MEDIA_IMAGES
Read audio	READ_EXTERNAL_STORAGE	READ_EXTERNAL_STORAGE	READ_MEDIA_AUDIO
Read video	READ_EXTERNAL_STORAGE	READ_EXTERNAL_STORAGE	READ_MEDIA_VIDEO
Write to gallery	WRITE_EXTERNAL_STORAGE	None needed	None needed
Background record	FOREGROUND_SERVICE	FOREGROUND_SERVICE	FOREGROUND_SERVICE
§J-2 — Canonical permission helper
// media/permissions/MediaPermissions.kt
object MediaPermissions {
    fun audioPermissions() = arrayOf(Manifest.permission.RECORD_AUDIO)
    fun cameraPermissions() = arrayOf(Manifest.permission.CAMERA)
    fun storageReadPermissions() = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        arrayOf(
            Manifest.permission.READ_MEDIA_IMAGES,
            Manifest.permission.READ_MEDIA_AUDIO,
            Manifest.permission.READ_MEDIA_VIDEO,
        )
    } else {
        arrayOf(Manifest.permission.READ_EXTERNAL_STORAGE)
    }
    fun areGranted(context: Context, vararg permissions: String): Boolean =
        permissions.all {
            ContextCompat.checkSelfPermission(context, it) == PackageManager.PERMISSION_GRANTED
        }
}
// Fragment usage — request and react
class RecordFragment : Fragment() {
    private val permissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { results ->
        val allGranted = results.values.all { it }
        if (allGranted) viewModel.startRecording()
        else showPermissionRationale()
    }
    private fun requestAudioPermissionAndRecord() {
        val perms = MediaPermissions.audioPermissions()
        if (MediaPermissions.areGranted(requireContext(), *perms)) {
            viewModel.startRecording()
        } else {
            permissionLauncher.launch(perms)
        }
    }
    private fun showPermissionRationale() {
        // Show a dialog explaining WHY the permission is needed with a retry option.
        // Never silently fail.
    }
}

§K — Error Handling & State Management
Unified media state model
// media/state/MediaState.kt
/** Top-level state for any media feature. Adapt T to the feature's result type. */
sealed class MediaState<out T> {
    data object Idle       : MediaState<Nothing>()
    data object Loading    : MediaState<Nothing>()
    data object Processing : MediaState<Nothing>()
    data class  Active<T>(val data: T)    : MediaState<T>()        // recording / playing
    data class  Paused<T>(val data: T)    : MediaState<T>()
    data class  Completed<T>(val result: T) : MediaState<T>()
    data class  Error(
        val message: String,
        val cause: Throwable? = null,
        val recoverable: Boolean = true,
    ) : MediaState<Nothing>()
}

Error mapping rules
Raw Exception / Code	Mapped State
IOException on file write	MediaState.Error("Storage full or unavailable", recoverable = true)
SecurityException	MediaState.Error("Permission denied — grant microphone access", recoverable = true)
PlaybackException.ERROR_CODE_IO_*	MediaState.Error("File not found or unreadable", recoverable = false)
SpeechRecognizer.ERROR_NO_MATCH	MediaState.Error("No speech detected — please try again", recoverable = true)
IllegalStateException on MediaRecorder	MediaState.Error("Recorder in invalid state — restart required", recoverable = false)
Audio focus permanent loss	MediaState.Paused(currentData) then MediaState.Idle
§L — Performance Audit Checklist
Threading
 All file I/O on Dispatchers.IO
 PCM processing on Dispatchers.Default
 UI updates only on Dispatchers.Main
 No runBlocking on the main thread
Memory
 MediaRecorder / AudioRecord / AudioTrack released immediately after use
 MediaMetadataRetriever always released in finally
 Bitmaps recycled after use; thumbnails scaled to display size before caching
 No static references to Context, Activity, or views from media objects
Camera
 ProcessCameraProvider.unbindAll() before rebinding
 Camera use cases limited to what is currently needed (do not bind ImageCapture when only previewing)
 setInitialTargetRotation set at build time to avoid rotation re-binds
ExoPlayer
 Single ExoPlayer instance reused across track changes; do not recreate per item
 player.release() only called in onDestroy, not onStop
 Background thread cache configured: SimpleCache with LeastRecentlyUsedCacheEvictor
Waveform rendering
 WaveformView.onDraw allocates zero objects — Paint, Path, RectF pre-allocated in init
 Amplitude list size capped to maxBars() — O(1) memory regardless of recording length
 invalidate() called only when data actually changed
§M — Security Guidelines
Requirement	Implementation
No raw file:// URIs shared externally	Use FileProvider (§I-3) — always
Validate media before processing	Check MIME type via MediaMetadataRetriever; reject unknown types
Temp files protected	Write to context.cacheDir (app-private); clean up in finally
No hardcoded paths	All paths constructed from context.cacheDir, context.filesDir, or MediaStore
Sensitive recordings	Store in context.filesDir (not external); encrypt with EncryptedFile from Security library if required
FileProvider scope	Grant FLAG_GRANT_READ_URI_PERMISSION per-share — never grant globally
Restrictions (Hard Limits)
The skill must never:

Block the main thread — no synchronous file I/O, no BitmapFactory.decode* on main
Leak media resources — every MediaRecorder, ExoPlayer, AudioRecord, AudioTrack, SpeechRecognizer, TextToSpeech, MediaMetadataRetriever, and MediaMuxer must be released
Use Environment.getExternalStorageDirectory() on API 29+
Expose raw file:// URIs to other applications
Hardcode file paths or directory names
Ignore audio focus changes during playback
Access ViewModels, repositories, or use-cases directly
Perform network requests (media download is the networking layer's responsibility)
Suppress or swallow exceptions — always map to a typed MediaState.Error
Use deprecated MediaPlayer for new playback code
