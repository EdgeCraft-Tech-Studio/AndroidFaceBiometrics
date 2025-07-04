# On-Device Face Recognition In Android 

> A simple Android app that performs on-device face recognition by comparing FaceNet embeddings against a vector database of user-given faces

## Goals

* Produce on-device face embeddings with FaceNet and use them to perform face recognition on a user-given set of images
* Store face-embedding and other metadata on-device and use vector-search to determine nearest-neighbors
* Use modern Android development practices and recommended architecture guidelines while maintaining code simplicity and modularity

## Setup

> Download the APK from the [Releases](https://github.com/shubham0204/OnDevice-Face-Recognition-Android/releases)

Clone the `main` branch,

```bash
$> git clone --depth=1 https://github.com/shubham0204/OnDevice-Face-Recognition-Android
```

Perform a Gradle sync, and run the application.

### Choosing the FaceNet model

The app provides two FaceNet models differing in the size of the embedding they provide. `facenet.tflite` outputs a 128-dimensional embedding and `facenet_512.tflite` a 512-dimensional embedding. In [FaceNet.kt](https://github.com/shubham0204/OnDevice-Face-Recognition-Android/blob/main/app/src/main/java/com/ml/shubham0204/facenet_android/domain/embeddings/FaceNet.kt), you may change the model by modifying the path of the TFLite model,

```kotlin
// facenet
interpreter =
    Interpreter(FileUtil.loadMappedFile(context, "facenet.tflite"), interpreterOptions)

// facenet-512
interpreter =
            Interpreter(FileUtil.loadMappedFile(context, "facenet_512.tflite"), interpreterOptions)
```

For change `embeddingDims` in the same file,

```kotlin
// facenet
private val embeddingDim = 128

// facenet-512
private val embeddingDim = 512
```

Then, in [DataModels.kt](https://github.com/shubham0204/OnDevice-Face-Recognition-Android/blob/main/app/src/main/java/com/ml/shubham0204/facenet_android/data/DataModels.kt), change the dimensions of the `faceEmbedding` attribute,

```kotlin
@Entity
data class FaceImageRecord(
    // primary-key of `FaceImageRecord`
    @Id var recordID: Long = 0,

    // personId is derived from `PersonRecord`
    @Index var personID: Long = 0,

    var personName: String = "",

    // the FaceNet-512 model provides a 512-dimensional embedding
    // the FaceNet model provides a 128-dimensional embedding
    @HnswIndex(dimensions = 512)
    var faceEmbedding: FloatArray = floatArrayOf()
)
```

## Working

![working](https://github.com/shubham0204/OnDevice-Face-Recognition-Android/assets/41076823/def3d020-e36a-44c6-b964-866786c36e3d)


We use the [FaceNet](https://arxiv.org/abs/1503.03832) model, which given a 160 * 160 cropped face image, produces an embedding of 128 or 512 elements capturing facial features that uniquely identify the face. We represent the embedding model as a function $M$ that accepts a cropped face image and returns a vector/embedding/list of FP numbers.

1. When users select an image, the app uses MLKit's `FaceDetector` to crop faces from the image. Each image is labelled with the person's name. See [`MLKitFaceDetector.kt`](https://github.com/shubham0204/OnDevice-Face-Recognition-Android/blob/main/app/src/main/java/com/ml/shubham0204/facenet_android/domain/face_detection/MLKitFaceDetector.kt).
2. Each cropped face is transformed into a vector/embedding with FaceNet. See [`FaceNet.kt`](https://github.com/shubham0204/OnDevice-Face-Recognition-Android/blob/main/app/src/main/java/com/ml/shubham0204/facenet_android/domain/embeddings/FaceNet.kt).
3. We store these face embeddings in a vector database, that enables a faster nearest-neighbor search.
4. Now, in the camera preview, for each frame, we perform face detection with MLKit's `FaceDetector` as in (1) and produce face embeddings for the face as in (2). We compare this face embedding (query vector) with those present in the vector database, and determines the name/label of the embedding (nearest-neighbor) closest to the query vector using cosine similarity.
5. The vector database performs a lossy compression on the embeddings stored in it, and hence the distance returned with the nearest-neighbor is also an estimate. Hence, we re-compute the cosine similarity between the nearest-neighbor vector and the query vector. See [`ImageVectorUseCase.kt`](https://github.com/shubham0204/OnDevice-Face-Recognition-Android/blob/main/app/src/main/java/com/ml/shubham0204/facenet_android/domain/ImageVectorUseCase.kt)

## Tools

1. [TensorFlow Lite](https://ai.google.dev/edge/lite) as a runtime to execute the FaceNet model
2. [Mediapipe Face Detection](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/android) to crop faces from the image
3. [ObjectBox](https://objectbox.io) for on-device vector-store and NoSQL database
