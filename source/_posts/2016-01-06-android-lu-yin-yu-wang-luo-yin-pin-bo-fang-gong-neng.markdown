---
layout: post
title: "Android - 录音与网络音频播放功能"
date: 2016-01-06 10:03:14 +0800
comments: true
categories: Android
---
Android - 录音与网络音频播放功能
--------

在美学的项目中，涉及到音频的录制与播放，其功能的实现具体步骤如下。
首先需要以下3个类：
	* PressToSpeakListener
	* VoicePlayManager
	* VoiceRecorder

在项目中使用上类实现的功能是设置两个个button（一个录音，一个播放），用户长按button可以实现录音功能，而当用户松开button时，结束录音。然后用户点击播放按钮可以实现播放的功能。具体实现只使用了以下代码
```java
private void initRecorder() {
    String output = String.format("reply_%d_%d", question.getId(), Calendar.getInstance().getTimeInMillis());
    mAddVoice.setOnTouchListener(new PressToSpeakListener(this, mRecordStatus, output, new PressToSpeakListener.Callback() {
        @Override
        public void onPressed() {
            Log.e(TAG, ">>>onPressed");
        }

        @Override
        public void onRelease() {
            Log.e(TAG, ">>>onRelease");
        }

        @Override
        public void onRecorded(int length) {
            Log.e(TAG, ">>>onRecorded");
            recordTime = length;
        }

        @Override
        public void onConverted(File output) {
            Log.e(TAG, ">>>onConverted");
            recordFile = output;
            mAddVoice.setVisibility(View.GONE);
            mAudioPlayLayout.setVisibility(View.VISIBLE);
            //此时实现播放录音的功能
            mVoicePlay.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    voicePlayManager.play(QuestionDetailActivity.this, null, null, recordFile.getAbsolutePath());
                }
            });
        }
    }));
}
```

下面给出3个相关类的代码：
VoiceRecorder.java
```java
public class VoiceRecorder {
    private final String TAG = VoiceRecorder.class.getSimpleName();
    public static final int WHAT_VOLUME = 1;

    private Context mContext;
    private AudioRecord mAudioRecord;
    private String mOutput;
    private short[] mBuffer;
    //    private MediaRecorder mMediaRecorder;
    private OnRecordingListener mListener;

    private File mPcm;
    private File mMp3;

    private boolean mIsRecording = false;
    private double mLastVolume = 0;
    private long startTime = 0;

    public VoiceRecorder(Context context, String output ,OnRecordingListener listener) {
        this.mContext = context;
        this.mOutput = output;
        this.mListener = listener;
        if (TextUtils.isEmpty(this.mOutput)) {
            this.mOutput = String.format("voice_%d", Calendar.getInstance().getTimeInMillis());
        }
        init();
    }

    public void init() {
        int bufferSize = AudioRecord.getMinBufferSize(Lame.SAMPLE_RATE, AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT);
        mBuffer = new short[bufferSize];
        mAudioRecord = new AudioRecord(MediaRecorder.AudioSource.MIC, Lame.SAMPLE_RATE, AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT, bufferSize);
        String dir = FileUtils.getFilesDir(mContext).getAbsolutePath() + File.separator + "record";
        File file = new File(dir);
        if (!file.exists()) file.mkdirs();
        mPcm = new File(dir + File.separator + mOutput + ".pcm");
        mMp3 = new File(dir + File.separator + mOutput + ".mp3");
    }


    public File getPcm() {
        return mPcm;
    }

    public File getMp3() {
        return mMp3;
    }

    /**
     * 开始录音
     */
    public void start() throws IOException {
        if (getPcm().exists()) {
            getPcm().delete();
            getMp3().delete();
        }

        Calendar c = Calendar.getInstance();
        startTime = 0;
        startTime = c.getTimeInMillis();
        mAudioRecord.startRecording();
        startBufferedWrite(getPcm());

//        mMediaRecorder = new MediaRecorder();
//        mMediaRecorder.setAudioSource(MediaRecorder.AudioSource.MIC);
//        mMediaRecorder.setOutputFormat(MediaRecorder.OutputFormat.RAW_AMR);
//        mMediaRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.AMR_NB);
//        mMediaRecorder.setAudioSamplingRate(Lame.SAMPLE_RATE);
//        mMediaRecorder.setOutputFile(mRecordFile.getPcm().getAbsolutePath());
//        mMediaRecorder.prepare();
//        mMediaRecorder.start();
        mIsRecording = true;
        if (mListener != null) mListener.startRecording();
//        mHandler.sendEmptyMessage(WHAT_VOLUME);
//        mHandler.post(mRunnable);
    }

    /**
     * 停止录音
     */
    public int stop() {
//        if (mMediaRecorder != null) {
//            mMediaRecorder.stop();
//            mMediaRecorder.release();
//            mMediaRecorder = null;
//        }
        mIsRecording = false;
        if (mListener != null) {
            mListener.stopRecording();
        }
        mAudioRecord.stop();

        mHandler.removeMessages(WHAT_VOLUME);
        long nowTime = Calendar.getInstance().getTimeInMillis();
        // 如果录制时间小于1秒，则返回0，并删除文件
        if ((nowTime - startTime) < 1000) {
            try {
                getPcm().delete();
                getMp3().delete();
            } catch (Exception e) { e.printStackTrace(); }
            return 0;
        } else {
            // 转换为 MP3
            if (getPcm().exists()) {
                new AsyncTask<Void, Void, Void>() {
                    @Override
                    protected Void doInBackground(Void... params) {
                        // 由于讯飞的录音功能和我自己写的配置不太一致，所以需要调用encodeFile 方法而不能encodeFile2
                        Lame.getDefault().initEncoder(Lame.NUM_CHANNELS, Lame.SAMPLE_RATE, Lame.BIT_RATE, Lame.MODE, Lame.QUALITY);
                        int result = Lame.getDefault().encodeFile(getPcm().getAbsolutePath(), getMp3().getAbsolutePath());
                        Lame.getDefault().destroyEncoder();
                        if (mListener != null) mListener.converted(getMp3());
                        return null;
                    }
                }.execute();
            }
        }
        return (int) ((nowTime - startTime) / 1000);
    }

    /**
     * 丢弃录音文件
     */
    public void discard() {
        if (mAudioRecord != null) {
            mAudioRecord.stop();
        }
        mIsRecording = false;
        mHandler.removeMessages(WHAT_VOLUME);
        // 删除记录文件
        if (getPcm().exists()) {
            getPcm().delete();
            getMp3().delete();
        }
        if (mListener != null) mListener.discardRecording();
    }

    private void startBufferedWrite(final File file) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                DataOutputStream output = null;
                try {
                    output = new DataOutputStream(new BufferedOutputStream(new FileOutputStream(file)));
                    while (mIsRecording) {
                        int readSize = mAudioRecord.read(mBuffer, 0, mBuffer.length);
                        int v = 0;
                        for (int i = 0; i < readSize; i++) {
                            output.writeShort(mBuffer[i]);
                            v += mBuffer[i] * mBuffer[i];
                        }
                        Message msg = new Message();
                        msg.what = WHAT_VOLUME;
                        msg.arg1 = (Math.abs((int) (v / (float) readSize) / 10000) >> 1);
                        mHandler.sendMessage(msg);
                    }
                } catch (IOException e) {
                    Toast.makeText(mContext, e.getMessage(), Toast.LENGTH_SHORT).show();
                } finally {
                    if (output != null) {
                        try {
                            output.flush();
                        } catch (IOException e) {
                            Toast.makeText(mContext, e.getMessage(), Toast.LENGTH_SHORT).show();
                        } finally {
                            try {
                                output.close();
                            } catch (IOException e) {
                                Toast.makeText(mContext, e.getMessage(), Toast.LENGTH_SHORT).show();
                            }
                        }
                    }
                }
            }
        }).start();
    }

    protected Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case WHAT_VOLUME:
                    double volume = msg.arg1;
                    if (!mIsRecording) return;
                    if (volume != mLastVolume) {
                        mLastVolume = volume;
                        if (mListener != null) mListener.onVolumeChanged(mLastVolume);
                    }
//                    msg = obtainMessage(WHAT_VOLUME);
//                    sendMessageDelayed(msg, 100);
                    break;
            }
        }
    };

    public interface OnRecordingListener {
        void startRecording();
        void stopRecording();
        void discardRecording();
        void converted(File outPut);
        void onVolumeChanged(double volume);
    }
}
```

VoicePlayManager.java
```java
public class VoicePlayManager {
    private final String TAG = VoicePlayManager.class.getSimpleName();

    private ImageView mCtrPlayButton;
    private TextView mCtrLength;
    private AnimationDrawable mPlayingAnimation;

    private String remoteUrl;

    private MediaPlayer mediaPlayer;
    private PlayCompleteCallback callback;

    public void play(final Context context, ImageView ctrPlayButton, TextView ctrLength, String url) {
        try {
            release();
            mPlayingAnimation = null;
        } catch (Exception e) {

        }
        this.mCtrPlayButton = ctrPlayButton;
        this.mCtrLength = ctrLength;
        this.remoteUrl = url;
        if (mediaPlayer != null) {
            release();
        }
        mediaPlayer = new MediaPlayer(context);
        mediaPlayer.setOnPreparedListener(new MediaPlayer.OnPreparedListener() {
            @Override
            public void onPrepared(MediaPlayer mp) {
                mp.setPlaybackSpeed(1.0f);
            }
        });
        mediaPlayer.setOnInfoListener(new MediaPlayer.OnInfoListener() {
            @Override
            public boolean onInfo(MediaPlayer mp, int what, int extra) {
                Log.d(TAG, ">>>onInfo");
                switch (what) {
                    case MediaPlayer.MEDIA_INFO_BUFFERING_START:
                        if (mp.isPlaying()) {
                            mp.pause();
                        }
                        break;
                    case MediaPlayer.MEDIA_INFO_BUFFERING_END:
                        startPlayingAnimation();
                        mp.start();
                        break;
                    case MediaPlayer.MEDIA_INFO_DOWNLOAD_RATE_CHANGED:
                        Log.e(TAG, "" + extra + "kb/s" + "  ");
                        break;
                }
                return true;
            }
        });
        mediaPlayer.setOnCompletionListener(new MediaPlayer.OnCompletionListener() {
            @Override
            public void onCompletion(MediaPlayer mp) {
                Log.d(TAG, ">>>onCompletion");
                release();
                //这里是音频播放完之后的回调
                if (callback == null) {
                    callback = (PlayCompleteCallback) context;
                }
                callback.onPlayComplete();
            }
        });
        mediaPlayer.setOnErrorListener(new MediaPlayer.OnErrorListener() {
            @Override
            public boolean onError(MediaPlayer mp, int what, int extra) {
                release();
                return false;
            }
        });
        try {
            mediaPlayer.setDataSource(remoteUrl);
            mediaPlayer.prepareAsync();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void startPlayingAnimation() {
        if (mCtrPlayButton == null) return;
        if (mPlayingAnimation == null) {
            try {
                mPlayingAnimation = (AnimationDrawable) mCtrPlayButton.getDrawable();
            } catch (ClassCastException e) {
            }
        }
        if (mPlayingAnimation != null) {
            mPlayingAnimation.start();
        }
    }

    public void stopPlayingAnimation() {
        if (mCtrPlayButton == null) return;
        if (mPlayingAnimation == null) {
            try {
                mPlayingAnimation = (AnimationDrawable) mCtrPlayButton.getDrawable();
            } catch (ClassCastException e) {
            }
        }
        // 停止动画，并显示动画最后一帧
        if (mPlayingAnimation != null) {
            mPlayingAnimation.stop();
            mPlayingAnimation.selectDrawable(0);
        }
    }

    public void release() {
        try {
            stopPlayingAnimation();
            if (mediaPlayer == null)
                return;
            mediaPlayer.stop();
            mediaPlayer.release();
            mediaPlayer = null;
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public interface PlayCompleteCallback {
        void onPlayComplete();
    }
}
```

PressToSpeakListener.java
```java
public class PressToSpeakListener implements View.OnTouchListener, VoiceRecorder.OnRecordingListener {
    private static final String TAG = PressToSpeakListener.class.getSimpleName();

    private VoiceRecorder mVoiceRecorder;
    private PowerManager.WakeLock mWakeLock;
    private Context mContext;
    private View mStatusView;
    private ImageView mStatusVolume;
    private TextView recordingHint;
    private Callback mCallback;

    //    private View recordingContainer;
//
    public PressToSpeakListener() {
    }

    public PressToSpeakListener(Context context, View status, String output) {
        this(context, status, output, null);
    }

    public PressToSpeakListener(Context context, View status, String output, Callback callback) {
        this.mContext = context;
        this.mStatusView = status;
        this.mCallback = callback;
        this.mVoiceRecorder = new VoiceRecorder(mContext, output, this);
        this.mStatusVolume = ViewUtils.find(mStatusView, R.id.recording_image);
        this.recordingHint = ViewUtils.find(mStatusView, R.id.recording_hint);
    }

    //
    @Override
    public boolean onTouch(View v, MotionEvent event) {
        if (mWakeLock == null) {
            mWakeLock = ((PowerManager) mContext.getSystemService(Context.POWER_SERVICE)).newWakeLock(PowerManager.SCREEN_DIM_WAKE_LOCK, "recording");
        }
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                try {
                    v.setPressed(true);
                    mWakeLock.acquire();
                    mVoiceRecorder.start();
                    // TODO: 开始录音，显示录音提醒 View
                    mStatusView.setVisibility(View.VISIBLE);

                    ((Activity) mContext).runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            if (mCallback != null) mCallback.onPressed();
                        }
                    });
                } catch (Exception e) {
                    e.printStackTrace();
                    v.setPressed(false);
                    if (mWakeLock.isHeld()) {
                        mWakeLock.release();
                    }
                    // TODO: 录音失败，隐藏 View
                    mStatusView.setVisibility(View.INVISIBLE);
//                    Toast.makeText(ChatActivity.this, "录制失败", Toast.LENGTH_SHORT).show();
                    return false;
                }

                return true;
            case MotionEvent.ACTION_MOVE: {
                if (recordingHint != null) {
                    if (event.getY() < 0) {
                        recordingHint.setText(mContext.getString(R.string.release_to_cancel));
//                        recordingHint.setBackgroundResource(R.drawable.recording_text_hint_bg);
                    } else {
                        recordingHint.setText(mContext.getString(R.string.move_up_to_cancel));
//                        recordingHint.setBackgroundColor(Color.TRANSPARENT);
                    }
                }
                return true;
            }
            case MotionEvent.ACTION_UP:
                v.setPressed(false);
                // TODO:录音状态
                mStatusView.setVisibility(View.INVISIBLE);
                if (mWakeLock.isHeld())
                    mWakeLock.release();
                if (event.getY() < 0) {
                    // discard the recorded audio.
                    mVoiceRecorder.discard();
                } else {
                    // stop recording and send voice file
                    try {
                        int length = mVoiceRecorder.stop();
//                        if (length > 0) {
////                            sendVoice(mVoiceRecorder.getVoiceFilePath(), mVoiceRecorder.getVoiceFileName(toUsername), Integer.toString(length), false);
//                        } else {
//                            //Toast.makeText(getApplicationContext(), "录音时间太短", Toast.LENGTH_SHORT).show();
//                        }
                        if (mCallback != null) mCallback.onRecorded(length);
                    } catch (Exception e) {
                        e.printStackTrace();
//                        Toast.makeText(ChatActivity.this, "发送失败，请检测服务器是否连接", Toast.LENGTH_SHORT).show();
                    }

                }

                ((Activity) mContext).runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        if (mCallback != null) mCallback.onRelease();
                    }
                });
                return true;
            default:
                return false;
        }
    }

    @Override
    public void startRecording() {

    }

    @Override
    public void stopRecording() {

    }

    @Override
    public void discardRecording() {

    }

    @Override
    public void converted(final File output) {
        ((Activity) mContext).runOnUiThread(new Runnable() {
            @Override
            public void run() {
                if (mCallback != null) mCallback.onConverted(output);
            }
        });
    }

    @Override
    public void onVolumeChanged(double volume) {
        if (D) Log.e(TAG, "音量：" + volume);
        if (mStatusVolume == null) return;
        if (volume <= 33) {
            mStatusVolume.setImageResource(R.drawable.photo_record_notice_1);
        } else if (volume <= 66) {
            mStatusVolume.setImageResource(R.drawable.photo_record_notice_2);
        } else if (volume <= 100) {
            mStatusVolume.setImageResource(R.drawable.photo_record_notice_3);
        } else {
            mStatusVolume.setImageResource(R.drawable.photo_record_notice_1);
        }
    }

    public interface Callback {
        void onPressed();
        void onRelease();
        void onRecorded(int length);
        void onConverted(File output);
    }
}

```
