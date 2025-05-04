# Project3_Lab2
软件项目研发实践（3）Lab2
# 创建项目
首先创建一个新项目，选择“Empty Activity”。
![image](https://github.com/user-attachments/assets/820565c5-bc09-4e57-8ded-43a7c8c2fd1a)
将项目命名为“CameraXApp”，软件包名称更改为“com.android.example.cameraxapp”。选择Kotlin语言开发，设定最低支持的API Level 24。
![image](https://github.com/user-attachments/assets/bf13d46b-a0a8-4b70-b0af-9872502b3afc)
# 添加 Gradle 依赖
打开项目的模块（Module）的build.gradle 文件，并添加 CameraX 依赖项：
![image](https://github.com/user-attachments/assets/daae1183-08f8-4b73-b2f0-846b5724acd4)
# 创建项目布局
本项目中，涉及以下的功能：

CameraX PreviewView（用于预览相机图片/视频）。

用于控制图片拍摄的标准按钮。

用于开始/停止视频拍摄的标准按钮。

用于放置 2 个按钮的垂直指南。

打开res/layout/activity_main.xml 的 activity_main 布局文件，并将其替换为以下代码。

![image](https://github.com/user-attachments/assets/ddf3770c-a3ab-4b49-8c13-19b418304af5)
![image](https://github.com/user-attachments/assets/def60cce-3762-4f75-86e4-037f449ba5c6)
# 编写 MainActivity.kt 代码
<pre><code>```kotlin 
class MainActivity : AppCompatActivity() {
   private lateinit var viewBinding: ActivityMainBinding

   private var imageCapture: ImageCapture? = null

   private var videoCapture: VideoCapture<Recorder>? = null
   private var recording: Recording? = null

   private lateinit var cameraExecutor: ExecutorService

   override fun onCreate(savedInstanceState: Bundle?) {
       super.onCreate(savedInstanceState)
       viewBinding = ActivityMainBinding.inflate(layoutInflater)
       setContentView(viewBinding.root)

       // Request camera permissions
       if (allPermissionsGranted()) {
           startCamera()
       } else {
           ActivityCompat.requestPermissions(
               this, REQUIRED_PERMISSIONS, REQUEST_CODE_PERMISSIONS)
       }

       // Set up the listeners for take photo and video capture buttons
       viewBinding.imageCaptureButton.setOnClickListener { takePhoto() }
       viewBinding.videoCaptureButton.setOnClickListener { captureVideo() }

       cameraExecutor = Executors.newSingleThreadExecutor()
   }

   private fun takePhoto() {}

   private fun captureVideo() {}

   private fun startCamera() {}

   private fun allPermissionsGranted() = REQUIRED_PERMISSIONS.all {
       ContextCompat.checkSelfPermission(
           baseContext, it) == PackageManager.PERMISSION_GRANTED
   }

   override fun onDestroy() {
       super.onDestroy()
       cameraExecutor.shutdown()
   }

   companion object {
       private const val TAG = "CameraXApp"
       private const val FILENAME_FORMAT = "yyyy-MM-dd-HH-mm-ss-SSS"
       private const val REQUEST_CODE_PERMISSIONS = 10
       private val REQUIRED_PERMISSIONS =
           mutableListOf (
               Manifest.permission.CAMERA,
               Manifest.permission.RECORD_AUDIO
           ).apply {
               if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.P) {
                   add(Manifest.permission.WRITE_EXTERNAL_STORAGE)
               }
           }.toTypedArray()
   }
}
```</code></pre>
![image](https://github.com/user-attachments/assets/8b0066b8-8bc5-4dc7-af86-ed54d7593dfe)
# 请求必要的权限
应用需要获得用户授权才能打开相机；录制音频也需要麦克风权限；在 Android 9 § 及更低版本上，MediaStore 需要外部存储空间写入权限。在此步骤中，我们将实现这些必要的权限。

打开 AndroidManifest.xml，然后将以下代码行添加到 application 标记之前。
![image](https://github.com/user-attachments/assets/6ff39e1a-2d1a-48e6-98ec-a806d14b9b02)
![image](https://github.com/user-attachments/assets/4783758b-f295-45a1-bc4c-6e697f36d537)
![image](https://github.com/user-attachments/assets/98ee938e-a78f-4d2a-bf84-89feb760fa02)
![image](https://github.com/user-attachments/assets/6dbdc40f-1a46-4686-a307-c7ae13c0a930)
# 实现 Preview 用例
在相机应用中，取景器用于让用户预览他们拍摄的照片。我们将使用 CameraX Preview 类实现取景器。

如需使用 Preview，首先需要定义一个配置，然后系统会使用该配置创建用例的实例。生成的实例就是绑定到 CameraX 生命周期的内容。填充之前的startCamera() 函数
<pre><code>```kotlin 
private fun startCamera() {
   val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

   cameraProviderFuture.addListener({
       // Used to bind the lifecycle of cameras to the lifecycle owner
       val cameraProvider: ProcessCameraProvider = cameraProviderFuture.get()

       // Preview
       val preview = Preview.Builder()
          .build()
          .also {
              it.setSurfaceProvider(viewBinding.viewFinder.surfaceProvider)
          }

       // Select back camera as a default
       val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

       try {
           // Unbind use cases before rebinding
           cameraProvider.unbindAll()

           // Bind use cases to camera
           cameraProvider.bindToLifecycle(
               this, cameraSelector, preview)

       } catch(exc: Exception) {
           Log.e(TAG, "Use case binding failed", exc)
       }

   }, ContextCompat.getMainExecutor(this))
}
```</code></pre>
![image](https://github.com/user-attachments/assets/499e0449-a54b-4391-bf53-61f39269326e)
# 实现 ImageCapture 用例（拍照功能）
<pre><code>
  private fun takePhoto() {
   // Get a stable reference of the modifiable image capture use case
   val imageCapture = imageCapture ?: return

   // Create time stamped name and MediaStore entry.
   val name = SimpleDateFormat(FILENAME_FORMAT, Locale.US)
              .format(System.currentTimeMillis())
   val contentValues = ContentValues().apply {
       put(MediaStore.MediaColumns.DISPLAY_NAME, name)
       put(MediaStore.MediaColumns.MIME_TYPE, "image/jpeg")
       if(Build.VERSION.SDK_INT > Build.VERSION_CODES.P) {
           put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/CameraX-Image")
       }
   }

   // Create output options object which contains file + metadata
   val outputOptions = ImageCapture.OutputFileOptions
           .Builder(contentResolver,
                    MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
                    contentValues)
           .build()

   // Set up image capture listener, which is triggered after photo has
   // been taken
   imageCapture.takePicture(
       outputOptions,
       ContextCompat.getMainExecutor(this),
       object : ImageCapture.OnImageSavedCallback {
           override fun onError(exc: ImageCaptureException) {
               Log.e(TAG, "Photo capture failed: ${exc.message}", exc)
           }

           override fun
               onImageSaved(output: ImageCapture.OutputFileResults){
               val msg = "Photo capture succeeded: ${output.savedUri}"
               Toast.makeText(baseContext, msg, Toast.LENGTH_SHORT).show()
               Log.d(TAG, msg)
           }
       }
   )
}
</code></pre>
![image](https://github.com/user-attachments/assets/5f6c4fcb-4701-42e2-ad21-89b5d8f4f155)
![image](https://github.com/user-attachments/assets/f09f9966-a4d1-4d0d-9dee-73d19711445a)
# 实现 ImageAnalysis 用例
使用 ImageAnalysis 功能可让相机应用变得更加有趣。它允许定义实现 ImageAnalysis.Analyzer 接口的自定义类，并使用传入的相机帧调用该类。无需管理相机会话状态，甚至无需处理图像；与其他生命周期感知型组件一样，仅绑定到应用所需的生命周期就足够了。

将此分析器添加为 MainActivity.kt 中的内部类。分析器会记录图像的平均亮度。如需创建分析器，我们会替换实现 ImageAnalysis.Analyzer 接口的类中的 analyze 函数。

<pre><code>
  private class LuminosityAnalyzer(private val listener: LumaListener) : ImageAnalysis.Analyzer {

   private fun ByteBuffer.toByteArray(): ByteArray {
       rewind()    // Rewind the buffer to zero
       val data = ByteArray(remaining())
       get(data)   // Copy the buffer into a byte array
       return data // Return the byte array
   }

   override fun analyze(image: ImageProxy) {

       val buffer = image.planes[0].buffer
       val data = buffer.toByteArray()
       val pixels = data.map { it.toInt() and 0xFF }
       val luma = pixels.average()

       listener(luma)

       image.close()
   }
} 
</code></pre>
![image](https://github.com/user-attachments/assets/320e6620-fb6b-4a66-836b-e985d88112ea)
# 实现 VideoCapture 用例（拍摄视频）
<pre><code>
  // Implements VideoCapture use case, including start and stop capturing.
private fun captureVideo() {
   val videoCapture = this.videoCapture ?: return

   viewBinding.videoCaptureButton.isEnabled = false

   val curRecording = recording
   if (curRecording != null) {
       // Stop the current recording session.
       curRecording.stop()
       recording = null
       return
   }

   // create and start a new recording session
   val name = SimpleDateFormat(FILENAME_FORMAT, Locale.US)
              .format(System.currentTimeMillis())
   val contentValues = ContentValues().apply {
       put(MediaStore.MediaColumns.DISPLAY_NAME, name)
       put(MediaStore.MediaColumns.MIME_TYPE, "video/mp4")
       if (Build.VERSION.SDK_INT > Build.VERSION_CODES.P) {
           put(MediaStore.Video.Media.RELATIVE_PATH, "Movies/CameraX-Video")
       }
   }

   val mediaStoreOutputOptions = MediaStoreOutputOptions
       .Builder(contentResolver, MediaStore.Video.Media.EXTERNAL_CONTENT_URI)
       .setContentValues(contentValues)
       .build()
   recording = videoCapture.output
       .prepareRecording(this, mediaStoreOutputOptions)
       .apply {
           if (PermissionChecker.checkSelfPermission(this@MainActivity,
                   Manifest.permission.RECORD_AUDIO) ==
               PermissionChecker.PERMISSION_GRANTED)
           {
               withAudioEnabled()
           }
       }
       .start(ContextCompat.getMainExecutor(this)) { recordEvent ->
           when(recordEvent) {
               is VideoRecordEvent.Start -> {
                   viewBinding.videoCaptureButton.apply {
                       text = getString(R.string.stop_capture)
                       isEnabled = true
                   }
               }
               is VideoRecordEvent.Finalize -> {
                   if (!recordEvent.hasError()) {
                       val msg = "Video capture succeeded: " +
                           "${recordEvent.outputResults.outputUri}"
                       Toast.makeText(baseContext, msg, Toast.LENGTH_SHORT)
                            .show()
                       Log.d(TAG, msg)
                   } else {
                       recording?.close()
                       recording = null
                       Log.e(TAG, "Video capture ends with error: " +
                           "${recordEvent.error}")
                   }
                   viewBinding.videoCaptureButton.apply {
                       text = getString(R.string.start_capture)
                       isEnabled = true
                   }
               }
           }
       }
}
</code></pre>
![image](https://github.com/user-attachments/assets/168ab091-d649-44a6-a130-b549420b1780)


