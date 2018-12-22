


> Written with [StackEdit](https://stackedit.io/).
# Introduction

We will analyze the Android Audio system from four levels: 

1. AudioTrack
2. AudioRecord
3. AudioFlinger
4. AudioPolicy
5. Audio HAL

![](https://img-blog.csdn.net/20180812210335745)

# AudioTrack Testing Code

- frameworks/base/media/tests/MediaFrameworkTest/src/com/android/mediaframeworktest/functional/audio/MediaAudioTrackTest.java

```java
    //Test case 3: setStereoVolume() with mid volume returns SUCCESS
    @LargeTest
    public void testSetStereoVolumeMid() throws Exception {
        // constants for test
        final String TEST_NAME = "testSetStereoVolumeMid";
        final int TEST_SR = 22050;
        final int TEST_CONF = AudioFormat.CHANNEL_OUT_STEREO;
        final int TEST_FORMAT = AudioFormat.ENCODING_PCM_16BIT;
        final int TEST_MODE = AudioTrack.MODE_STREAM;
        final int TEST_STREAM_TYPE = AudioManager.STREAM_MUSIC;

        //-------- initialization --------------
        int minBuffSize = AudioTrack.getMinBufferSize(TEST_SR, TEST_CONF, TEST_FORMAT);
        AudioTrack track = new AudioTrack(TEST_STREAM_TYPE, TEST_SR, TEST_CONF, TEST_FORMAT, minBuffSize, TEST_MODE);
        byte data[] = new byte[minBuffSize/2];
        //--------    test        --------------
        track.write(data, 0, data.length);
        track.write(data, 0, data.length);
        track.play();
        float midVol = (AudioTrack.getMaxVolume() - AudioTrack.getMinVolume()) / 2;
        assertTrue(TEST_NAME, track.setStereoVolume(midVol, midVol) == AudioTrack.SUCCESS);
        //-------- tear down      --------------
        track.release();
    }
```

## AudioTrack.getMinBufferSize()

- frameworks/base/media/java/android/media/AudioTrack.java

```java
    /**
     * Returns the estimated minimum buffer size required for an AudioTrack
     * object to be created in the {@link #MODE_STREAM} mode.
     * The size is an estimate because it does not consider either the route or the sink,
     * since neither is known yet.  Note that this size doesn't
     * guarantee a smooth playback under load, and higher values should be chosen according to
     * the expected frequency at which the buffer will be refilled with additional data to play.
     * For example, if you intend to dynamically set the source sample rate of an AudioTrack
     * to a higher value than the initial source sample rate, be sure to configure the buffer size
     * based on the highest planned sample rate.
     * @param sampleRateInHz the source sample rate expressed in Hz.
     *   {@link AudioFormat#SAMPLE_RATE_UNSPECIFIED} is not permitted.
     * @param channelConfig describes the configuration of the audio channels.
     *   See {@link AudioFormat#CHANNEL_OUT_MONO} and
     *   {@link AudioFormat#CHANNEL_OUT_STEREO}
     * @param audioFormat the format in which the audio data is represented.
     *   See {@link AudioFormat#ENCODING_PCM_16BIT} and
     *   {@link AudioFormat#ENCODING_PCM_8BIT},
     *   and {@link AudioFormat#ENCODING_PCM_FLOAT}.
     * @return {@link #ERROR_BAD_VALUE} if an invalid parameter was passed,
     *   or {@link #ERROR} if unable to query for output properties,
     *   or the minimum buffer size expressed in bytes.
     */
    static public int getMinBufferSize(int sampleRateInHz, int channelConfig, int audioFormat) {
        int channelCount = 0;
        switch(channelConfig) {
        case AudioFormat.CHANNEL_OUT_MONO:
        case AudioFormat.CHANNEL_CONFIGURATION_MONO:
            channelCount = 1;
            break;
        case AudioFormat.CHANNEL_OUT_STEREO:
        case AudioFormat.CHANNEL_CONFIGURATION_STEREO:
            channelCount = 2;
            break;
        default:
            if (!isMultichannelConfigSupported(channelConfig)) {
                loge("getMinBufferSize(): Invalid channel configuration.");
                return ERROR_BAD_VALUE;
            } else {
                channelCount = AudioFormat.channelCountFromOutChannelMask(channelConfig);
            }
        }

        if (!AudioFormat.isPublicEncoding(audioFormat)) {
            loge("getMinBufferSize(): Invalid audio format.");
            return ERROR_BAD_VALUE;
        }

        // sample rate, note these values are subject to change
        // Note: AudioFormat.SAMPLE_RATE_UNSPECIFIED is not allowed
        if ( (sampleRateInHz < AudioFormat.SAMPLE_RATE_HZ_MIN) ||
                (sampleRateInHz > AudioFormat.SAMPLE_RATE_HZ_MAX) ) {
            loge("getMinBufferSize(): " + sampleRateInHz + " Hz is not a supported sample rate.");
            return ERROR_BAD_VALUE;
        }

        int size = native_get_min_buff_size(sampleRateInHz, channelCount, audioFormat);
        if (size <= 0) {
            loge("getMinBufferSize(): error querying hardware");
            return ERROR;
        }
        else {
            return size;
        }
    }
```

The `native_get_min_buff_size()` is from `frameworks/base/core/jni/android_media_AudioTrack.cpp` which maps to `android_media_AudioTrack_get_min_buff_size()` as below:

```cpp
// ----------------------------------------------------------------------------
// returns the minimum required size for the successful creation of a streaming AudioTrack
// returns -1 if there was an error querying the hardware.
static jint android_media_AudioTrack_get_min_buff_size(JNIEnv *env,  jobject thiz,
    jint sampleRateInHertz, jint channelCount, jint audioFormat) {

    size_t frameCount;
    const status_t status = AudioTrack::getMinFrameCount(&frameCount, AUDIO_STREAM_DEFAULT,
            sampleRateInHertz);
    if (status != NO_ERROR) {
        ALOGE("AudioTrack::getMinFrameCount() for sample rate %d failed with status %d",
                sampleRateInHertz, status);
        return -1;
    }
    const audio_format_t format = audioFormatToNative(audioFormat);
    if (audio_has_proportional_frames(format)) {
        const size_t bytesPerSample = audio_bytes_per_sample(format);
        return frameCount * channelCount * bytesPerSample;
    } else {
        return frameCount;
    }
}
```

- AudioTrack::getMinFrameCount()
	- frameworks/av/media/libaudioclient/AudioTrack.cpp

```cpp
// static
status_t AudioTrack::getMinFrameCount(
        size_t* frameCount,
        audio_stream_type_t streamType,
        uint32_t sampleRate)
{
    if (frameCount == NULL) {
        return BAD_VALUE;
    }

    // FIXME handle in server, like createTrack_l(), possible missing info:
    //          audio_io_handle_t output
    //          audio_format_t format
    //          audio_channel_mask_t channelMask
    //          audio_output_flags_t flags (FAST)
    uint32_t afSampleRate;
    status_t status;
    status = AudioSystem::getOutputSamplingRate(&afSampleRate, streamType);
    if (status != NO_ERROR) {
        ALOGE("Unable to query output sample rate for stream type %d; status %d",
                streamType, status);
        return status;
    }
    size_t afFrameCount;
    status = AudioSystem::getOutputFrameCount(&afFrameCount, streamType);
    if (status != NO_ERROR) {
        ALOGE("Unable to query output frame count for stream type %d; status %d",
                streamType, status);
        return status;
    }
    uint32_t afLatency;
    status = AudioSystem::getOutputLatency(&afLatency, streamType);
    if (status != NO_ERROR) {
        ALOGE("Unable to query output latency for stream type %d; status %d",
                streamType, status);
        return status;
    }

    // When called from createTrack, speed is 1.0f (normal speed).
    // This is rechecked again on setting playback rate (TODO: on setting sample rate, too).
    *frameCount = AudioSystem::calculateMinFrameCount(afLatency, afFrameCount, afSampleRate,
                                              sampleRate, 1.0f /*, 0 notificationsPerBufferReq*/);

    // The formula above should always produce a non-zero value under normal circumstances:
    // AudioTrack.SAMPLE_RATE_HZ_MIN <= sampleRate <= AudioTrack.SAMPLE_RATE_HZ_MAX.
    // Return error in the unlikely event that it does not, as that's part of the API contract.
    if (*frameCount == 0) {
        ALOGE("AudioTrack::getMinFrameCount failed for streamType %d, sampleRate %u",
                streamType, sampleRate);
        return BAD_VALUE;
    }
    ALOGV("getMinFrameCount=%zu: afFrameCount=%zu, afSampleRate=%u, afLatency=%u",
            *frameCount, afFrameCount, afSampleRate, afLatency);
    return NO_ERROR;
}
```

- audioFormatToNative()
	- frameworks/base/core/jni/android_media_AudioFormat.h

```cpp
static inline audio_format_t audioFormatToNative(int audioFormat)
{
    switch (audioFormat) {
    case ENCODING_PCM_16BIT:
        return AUDIO_FORMAT_PCM_16_BIT;
    case ENCODING_PCM_8BIT:
        return AUDIO_FORMAT_PCM_8_BIT;
    case ENCODING_PCM_FLOAT:
        return AUDIO_FORMAT_PCM_FLOAT;
    case ENCODING_AC3:
        return AUDIO_FORMAT_AC3;
    case ENCODING_E_AC3:
        return AUDIO_FORMAT_E_AC3;
    case ENCODING_DTS:
        return AUDIO_FORMAT_DTS;
    case ENCODING_DTS_HD:
        return AUDIO_FORMAT_DTS_HD;
    case ENCODING_MP3:
        return AUDIO_FORMAT_MP3;
    case ENCODING_AAC_LC:
        return AUDIO_FORMAT_AAC_LC;
    case ENCODING_AAC_HE_V1:
        return AUDIO_FORMAT_AAC_HE_V1;
    case ENCODING_AAC_HE_V2:
        return AUDIO_FORMAT_AAC_HE_V2;
    case ENCODING_DOLBY_TRUEHD:
        return AUDIO_FORMAT_DOLBY_TRUEHD;
    case ENCODING_IEC61937:
        return AUDIO_FORMAT_IEC61937;
    case ENCODING_AAC_ELD:
        return AUDIO_FORMAT_AAC_ELD;
    case ENCODING_AAC_XHE:
        return AUDIO_FORMAT_AAC_XHE;
    case ENCODING_AC4:
        return AUDIO_FORMAT_AC4;
    case ENCODING_E_AC3_JOC:
        return AUDIO_FORMAT_E_AC3_JOC;
    case ENCODING_DEFAULT:
        return AUDIO_FORMAT_DEFAULT;
    default:
        return AUDIO_FORMAT_INVALID;
    }
}
```

- audio_has_proportional_frames()
	- system/media/audio/include/system/audio.h

```cpp
/**
 * For this format, is the number of PCM audio frames directly proportional
 * to the number of data bytes?
 *
 * In other words, is the format transported as PCM audio samples,
 * but not necessarily scalable or mixable.
 * This returns true for real PCM, but also for AUDIO_FORMAT_IEC61937,
 * which is transported as 16 bit PCM audio, but where the encoded data
 * cannot be mixed or scaled.
 */
static inline bool audio_has_proportional_frames(audio_format_t format)
{
    audio_format_t mainFormat = audio_get_main_format(format);
    return (mainFormat == AUDIO_FORMAT_PCM
            || mainFormat == AUDIO_FORMAT_IEC61937);
}

/**
 * Extract the primary format, eg. PCM, AC3, etc.
 */
static inline audio_format_t audio_get_main_format(audio_format_t format)
{
    return (audio_format_t)(format & AUDIO_FORMAT_MAIN_MASK);
}
```

- audio_bytes_per_sample()
	- system/media/audio/include/system/audio.h

```cpp
static inline size_t audio_bytes_per_sample(audio_format_t format)
{
    size_t size = 0;

    switch (format) {
    case AUDIO_FORMAT_PCM_32_BIT:
    case AUDIO_FORMAT_PCM_8_24_BIT:
        size = sizeof(int32_t);
        break;
    case AUDIO_FORMAT_PCM_24_BIT_PACKED:
        size = sizeof(uint8_t) * 3;
        break;
    case AUDIO_FORMAT_PCM_16_BIT:
    case AUDIO_FORMAT_IEC61937:
        size = sizeof(int16_t);
        break;
    case AUDIO_FORMAT_PCM_8_BIT:
        size = sizeof(uint8_t);
        break;
    case AUDIO_FORMAT_PCM_FLOAT:
        size = sizeof(float);
        break;
    default:
        break;
    }
    return size;
}
```
## AudioSystem::getOutputSamplingRate()

```cpp
status_t AudioSystem::getOutputSamplingRate(uint32_t* samplingRate, audio_stream_type_t streamType)
{
    audio_io_handle_t output;

    if (streamType == AUDIO_STREAM_DEFAULT) {
        streamType = AUDIO_STREAM_MUSIC;
    }

    output = getOutput(streamType);
    if (output == 0) {
        return PERMISSION_DENIED;
    }

    return getSamplingRate(output, samplingRate);
}

audio_io_handle_t AudioSystem::getOutput(audio_stream_type_t stream)
{
    const sp<IAudioPolicyService>& aps = AudioSystem::get_audio_policy_service();
    if (aps == 0) return 0;
    return aps->getOutput(stream);
}

// client singleton for AudioPolicyService binder interface
// protected by gLockAPS
sp<IAudioPolicyService> AudioSystem::gAudioPolicyService;
sp<AudioSystem::AudioPolicyServiceClient> AudioSystem::gAudioPolicyServiceClient;

// establish binder interface to AudioPolicy service
const sp<IAudioPolicyService> AudioSystem::get_audio_policy_service()
{
    sp<IAudioPolicyService> ap;
    sp<AudioPolicyServiceClient> apc;
    {
        Mutex::Autolock _l(gLockAPS);
        if (gAudioPolicyService == 0) {
            sp<IServiceManager> sm = defaultServiceManager();
            sp<IBinder> binder;
            do {
                binder = sm->getService(String16("media.audio_policy"));
                if (binder != 0)
                    break;
                ALOGW("AudioPolicyService not published, waiting...");
                usleep(500000); // 0.5 s
            } while (true);
            if (gAudioPolicyServiceClient == NULL) {
                gAudioPolicyServiceClient = new AudioPolicyServiceClient();
            }
            binder->linkToDeath(gAudioPolicyServiceClient);
            gAudioPolicyService = interface_cast<IAudioPolicyService>(binder);
            LOG_ALWAYS_FATAL_IF(gAudioPolicyService == 0);
            apc = gAudioPolicyServiceClient;
            // Make sure callbacks can be received by gAudioPolicyServiceClient
            ProcessState::self()->startThreadPool();
        }
        ap = gAudioPolicyService;
    }
    if (apc != 0) {
        int64_t token = IPCThreadState::self()->clearCallingIdentity();
        ap->registerClient(apc);
        ap->setAudioPortCallbacksEnabled(apc->isAudioPortCbEnabled());
        IPCThreadState::self()->restoreCallingIdentity(token);
    }

    return ap;
}

status_t AudioSystem::getSamplingRate(audio_io_handle_t ioHandle,
                                      uint32_t* samplingRate)
{
    const sp<IAudioFlinger>& af = AudioSystem::get_audio_flinger();
    if (af == 0) return PERMISSION_DENIED;
    sp<AudioIoDescriptor> desc = getIoDescriptor(ioHandle);
    if (desc == 0) {
        *samplingRate = af->sampleRate(ioHandle);
    } else {
        *samplingRate = desc->mSamplingRate;
    }
    if (*samplingRate == 0) {
        ALOGE("AudioSystem::getSamplingRate failed for ioHandle %d", ioHandle);
        return BAD_VALUE;
    }

    ALOGV("getSamplingRate() ioHandle %d, sampling rate %u", ioHandle, *samplingRate);

    return NO_ERROR;
}

sp<AudioIoDescriptor> AudioSystem::getIoDescriptor(audio_io_handle_t ioHandle)
{
    sp<AudioIoDescriptor> desc;
    const sp<AudioFlingerClient> afc = getAudioFlingerClient();
    if (afc != 0) {
        desc = afc->getIoDescriptor(ioHandle);
    }
    return desc;
}

const sp<AudioSystem::AudioFlingerClient> AudioSystem::getAudioFlingerClient()
{
    // calling get_audio_flinger() will initialize gAudioFlingerClient if needed
    const sp<IAudioFlinger>& af = AudioSystem::get_audio_flinger();
    if (af == 0) return 0;
    Mutex::Autolock _l(gLock);
    return gAudioFlingerClient;
}

// establish binder interface to AudioFlinger service
const sp<IAudioFlinger> AudioSystem::get_audio_flinger()
{
    sp<IAudioFlinger> af;
    sp<AudioFlingerClient> afc;
    {
        Mutex::Autolock _l(gLock);
        if (gAudioFlinger == 0) {
            sp<IServiceManager> sm = defaultServiceManager();
            sp<IBinder> binder;
            do {
                binder = sm->getService(String16("media.audio_flinger"));
                if (binder != 0)
                    break;
                ALOGW("AudioFlinger not published, waiting...");
                usleep(500000); // 0.5 s
            } while (true);
            if (gAudioFlingerClient == NULL) {
                gAudioFlingerClient = new AudioFlingerClient();
            } else {
                if (gAudioErrorCallback) {
                    gAudioErrorCallback(NO_ERROR);
                }
            }
            binder->linkToDeath(gAudioFlingerClient);
            gAudioFlinger = interface_cast<IAudioFlinger>(binder);
            LOG_ALWAYS_FATAL_IF(gAudioFlinger == 0);
            afc = gAudioFlingerClient;
            // Make sure callbacks can be received by gAudioFlingerClient
            ProcessState::self()->startThreadPool();
        }
        af = gAudioFlinger;
    }
    if (afc != 0) {
        int64_t token = IPCThreadState::self()->clearCallingIdentity();
        af->registerClient(afc);
        IPCThreadState::self()->restoreCallingIdentity(token);
    }
    return af;
}
```

## AudioSystem::getOutputFrameCount

```cpp
status_t AudioSystem::getOutputFrameCount(size_t* frameCount, audio_stream_type_t streamType)
{
    audio_io_handle_t output;

    if (streamType == AUDIO_STREAM_DEFAULT) {
        streamType = AUDIO_STREAM_MUSIC;
    }

    output = getOutput(streamType);
    if (output == AUDIO_IO_HANDLE_NONE) {
        return PERMISSION_DENIED;
    }

    return getFrameCount(output, frameCount);
}

```
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTM3MDQ3NTQ0MSwtMzY1OTgwNDJdfQ==
-->