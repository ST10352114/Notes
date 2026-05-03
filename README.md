# LuckyCat — Study Notes for Lecturer Q&A

---

## PART 1: LOGIN & REGISTRATION LOGIC

---

### 1.1 The Big Picture — How Authentication Works

The app uses **Firebase Authentication** (email/password method). Firebase is Google's backend-as-a-service platform. Instead of building your own server to store usernames and passwords, Firebase handles all of that securely in the cloud. Your app just sends the email and password to Firebase, and Firebase tells you if it worked or not.

The authentication flow has **three layers** in this project:

```
UI Screen (what the user sees)
    ↓  calls
AuthViewModel (the logic brain)
    ↓  calls
Firebase Auth SDK (the cloud service)
```

---

### 1.2 AuthViewModel — The Brain of Authentication

**File:** `ui/screens/auth/AuthViewModel.kt`

```kotlin
class AuthViewModel : ViewModel() {
    private val auth = FirebaseAuth.getInstance()
    ...
}
```

- `AuthViewModel` extends `ViewModel`, which is an Android architecture component. A ViewModel **survives screen rotations** — so if the user tilts their phone during login, the auth state is not lost.
- `FirebaseAuth.getInstance()` gives you a singleton reference to Firebase's auth system. "Singleton" means there is only ever one instance of it — every part of the app shares the same Firebase connection.

**The `registerUser` function:**
```kotlin
fun registerUser(email, password, onSuccess, onError) {
    auth.createUserWithEmailAndPassword(email, password)
        .addOnCompleteListener { task ->
            if (task.isSuccessful) onSuccess()
            else onError(task.exception?.localizedMessage ?: "Unknown error")
        }
}
```
- `createUserWithEmailAndPassword` is a **Firebase SDK call** that sends the email and password to Firebase servers over the internet.
- It runs **asynchronously** — meaning it does not freeze the app while waiting for Firebase to respond. The `.addOnCompleteListener` block runs only once Firebase replies.
- `task.isSuccessful` is a boolean that tells you whether Firebase created the account successfully.
- If it fails (e.g. email already in use, password too weak), `task.exception?.localizedMessage` gives you a human-readable error message to show the user.

**The `loginUser` function:**
```kotlin
fun loginUser(email, password, onSuccess, onError) {
    auth.signInWithEmailAndPassword(email, password)
        .addOnCompleteListener { task ->
            if (task.isSuccessful) onSuccess()
            else onError(task.exception?.localizedMessage ?: "Login failed.")
        }
}
```
- Same pattern as register, but uses `signInWithEmailAndPassword` instead.
- Firebase checks the credentials against its database and responds with success or failure.
- Notice the functions take `onSuccess` and `onError` as **callback parameters** — these are functions passed in from the UI layer that tell the ViewModel what to do on each outcome. This keeps the ViewModel decoupled from navigation logic.

---

### 1.3 RegisterScreen — The UI Layer

**File:** `ui/screens/auth/RegisterScreen.kt`

**State variables (what drives the UI):**
```kotlin
var username by remember { mutableStateOf("") }
var email by remember { mutableStateOf("") }
var password by remember { mutableStateOf("") }
var errorMessage by remember { mutableStateOf("") }
```
- `remember { mutableStateOf(...) }` is Jetpack Compose's way of storing UI state. When these values change, Compose automatically redraws (re-composes) only the affected parts of the screen.
- The password field uses `PasswordVisualTransformation()` — this replaces each character with a dot so the password is hidden on screen.

**Local validation before Firebase is called:**
```kotlin
val hasMinLen = password.length >= 8
val hasUpper = password.any { it.isUpperCase() }
val hasNumber = password.any { it.isDigit() }

if (username.isBlank() || email.isBlank()) {
    errorMessage = "Please fill in all fields."
} else if (!hasMinLen || !hasUpper || !hasNumber) {
    errorMessage = "Password must contain at least 8 characters, 1 uppercase, and 1 number."
} else {
    onRegisterSubmit(email, username, password)
}
```
- This is **client-side validation** — it runs on the device *before* any network call is made. This saves unnecessary Firebase calls and gives instant feedback.
- `password.any { it.isUpperCase() }` uses a Kotlin lambda to iterate every character and check if any of them is uppercase.
- Only when all local checks pass does the app call `onRegisterSubmit`, which triggers the ViewModel, which calls Firebase.

**How the Register screen connects to the ViewModel (via Navigation):**

In `AppNavigation.kt`:
```kotlin
RegisterScreen(
    onNavigateToLogin = { navController.navigate("login") },
    onRegisterSubmit = { email, _, password ->
        authViewModel.registerUser(email, password,
            onSuccess = { navController.navigate("login") },
            onError = { /* handle error */ }
        )
    }
)
```
- The `_` in `{ email, _, password ->` discards the `username` parameter — Firebase Auth does not use a display name during basic registration, only email and password.
- On success, the user is navigated back to the Login screen to sign in with their new account.

---

### 1.4 LoginScreen — The UI Layer

**File:** `ui/screens/auth/LoginScreen.kt`

**What happens when the Log In button is tapped:**
```kotlin
onClick = {
    if (username.isBlank() || password.isBlank()) {
        errorMessage = "Please enter both email and password."
    } else {
        errorMessage = ""
        authViewModel.loginUser(
            email = username.trim(),
            password = password,
            onSuccess = { onNavigateToHome() },
            onError = { error -> errorMessage = error }
        )
    }
}
```
- `username.trim()` removes any leading/trailing spaces the user may have accidentally typed — prevents "user@email.com " from being treated as a different address.
- If Firebase login succeeds, `onNavigateToHome()` is called — this is a navigation lambda passed in from `AppNavigation`, which uses `navController.navigate(Screen.Home.route)` and pops the login screen off the back stack so the user cannot press Back to get back to it.
- If Firebase login fails, the error string from Firebase (e.g. "The password is incorrect") is put into `errorMessage`, which is a state variable that Compose will display in red on screen.

**Navigation back-stack management:**
```kotlin
navController.navigate(Screen.Home.route) { 
    popUpTo("login") { inclusive = true } 
}
```
- `popUpTo("login") { inclusive = true }` removes the login screen from the navigation back stack entirely. This means pressing the back button from Home will exit the app, not go back to the login screen.

---

### 1.5 Session Persistence — What Happens on App Relaunch

Firebase Auth **automatically persists the user's session** to the device. When the app is reopened, `FirebaseAuth.getInstance().currentUser` will be non-null if the user was previously logged in.

In `AppNavigation.kt` the Splash screen navigates to login, and the Login screen would check `currentUser` — this is how authenticated users can skip the login screen on reopen. Firebase handles the token refresh automatically behind the scenes.

---

### Likely Lecturer Questions on Auth

**Q: Why use a ViewModel instead of putting Firebase calls directly in the screen?**
A: Separation of concerns. The screen only handles UI. The ViewModel handles business logic. If Firebase is called from the screen directly, a screen rotation would cancel the call and the result would be lost.

**Q: What happens if the user's internet is off during login?**
A: Firebase's `addOnCompleteListener` will fire with `task.isSuccessful = false` and `task.exception` will contain a network error. The `onError` callback passes the error message to `errorMessage`, which Compose displays in red on screen.

**Q: Is the password stored anywhere in the app?**
A: No. The password is only held in the `password` state variable (in memory, in the Compose UI) for as long as the screen is active. It is passed directly to Firebase and never written to disk or to Room.

**Q: What is the difference between `createUserWithEmailAndPassword` and `signInWithEmailAndPassword`?**
A: `createUserWithEmailAndPassword` creates a brand new user account in Firebase. `signInWithEmailAndPassword` checks the credentials against an existing account. Both are async and return a Task.

---
---

## PART 2: RECEIPT UPLOAD & STORAGE LOGIC

---

### 2.1 The Big Picture — How Receipt Images Work

The app does **not** upload receipts to the cloud. Instead, images are:

1. Captured from the camera or picked from the gallery
2. Compressed in memory (as a `ByteArray`)
3. Stored directly in the **Room SQLite database** on the device as a binary blob

```
Camera / Gallery
    ↓
Raw bytes read from file
    ↓
Compressed to JPEG ByteArray (max 1024px, 60% quality)
    ↓
Stored in Room DB column: receiptImage BLOB
    ↓
Retrieved on demand when viewing expense details
```

---

### 2.2 Two Ways to Attach a Receipt

**File:** `ui/screens/Expenses/CreateExpenses.kt`

#### Path 1 — Camera

```kotlin
val cameraLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.TakePicture()
) { success: Boolean ->
    if (success) {
        cameraImageUri?.let { uri ->
            val stream = context.contentResolver.openInputStream(uri)
            val bytes = stream?.readBytes()
            val compressed = processAndCompressImage(bytes)
            receiptImageBytes = compressed
            expenseViewModel.receiptImageBytes = compressed
        }
    }
}
```

- `ActivityResultContracts.TakePicture()` is the modern Android way to launch the camera. It returns a boolean — `true` if the photo was taken, `false` if the user cancelled.
- Before launching the camera, the app must request the `CAMERA` permission at runtime:
```kotlin
val cameraPermissionLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.RequestPermission()
) { isGranted ->
    if (isGranted) { /* launch camera */ }
}
// When button tapped:
cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
```
- Permissions are requested first, then if granted, the camera is launched. This is required for Android 6.0 (API 23) and above.

**FileProvider — why it's needed:**
```kotlin
val uri = FileProvider.getUriForFile(
    context,
    "${context.packageName}.fileprovider",
    photoFile
)
```
- Android security rules prevent apps from sharing raw file paths with other apps (like the camera app). `FileProvider` creates a temporary, secure URI that the camera app can write the photo to, without exposing the actual file path.
- The photo is saved to the app's external files directory under `Pictures/`, using a temp file named with a timestamp: `JPEG_<timestamp>_.jpg`.

#### Path 2 — Gallery

```kotlin
val galleryLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.GetContent()
) { uri: Uri? ->
    uri?.let {
        val stream = context.contentResolver.openInputStream(it)
        val bytes = stream?.readBytes()
        val compressed = processAndCompressImage(bytes)
        receiptImageBytes = compressed
        expenseViewModel.receiptImageBytes = compressed
    }
}
// Launch with:
galleryLauncher.launch("image/*")
```
- `ActivityResultContracts.GetContent()` opens the device's file picker filtered to images only (`"image/*"`).
- The selected image is read as a stream via `ContentResolver` — this works regardless of where the image actually lives on the device (SD card, cloud, Downloads folder, etc.).
- No permission is needed on Android 13+ for picking from the gallery via this API.

---

### 2.3 Image Compression — Why and How

```kotlin
fun processAndCompressImage(bytes: ByteArray): ByteArray? {
    // Step 1: Read image dimensions only (no full decode)
    val options = BitmapFactory.Options().apply { inJustDecodeBounds = true }
    BitmapFactory.decodeByteArray(bytes, 0, bytes.size, options)

    // Step 2: Calculate how much to shrink (must be a power of 2)
    var inSampleSize = 1
    val targetSize = 1024
    while (options.outHeight / inSampleSize > targetSize || 
           options.outWidth / inSampleSize > targetSize) {
        inSampleSize *= 2
    }

    // Step 3: Decode at reduced size
    val decodeOptions = BitmapFactory.Options().apply { this.inSampleSize = inSampleSize }
    val bitmap = BitmapFactory.decodeByteArray(bytes, 0, bytes.size, decodeOptions)

    // Step 4: Re-compress to JPEG at 60% quality
    val out = ByteArrayOutputStream()
    bitmap.compress(Bitmap.CompressFormat.JPEG, 60, out)
    return out.toByteArray()
}
```

**Why compress?** Room's SQLite database has a **CursorWindow size limit of ~2MB**. A raw photo from a modern phone can be 5–10MB. Without compression, saving or reading the expense would crash with a `CursorWindowAllocationException`.

**How `inSampleSize` works:** Setting `inSampleSize = 2` makes Android load only 1 in every 2 pixels — halving the resolution. Setting it to 4 loads 1 in 4 pixels. It must be a power of 2 (1, 2, 4, 8...). The while loop doubles it until both dimensions are under 1024px.

**`inJustDecodeBounds = true`:** This tells Android to read only the image header (width/height/type) without loading the full pixel data into memory. This avoids an out-of-memory crash before we even know the dimensions.

**60% JPEG quality:** A well-chosen balance — visually the receipt is still readable, but the file size is drastically smaller. For a receipt image this is more than sufficient.

---

### 2.4 Storing the Image in Room

**The Expense data class (Room Entity):**
```kotlin
@Entity(tableName = "expenses")
data class Expense(
    @PrimaryKey val id: String,
    val amount: Double,
    val categoryId: String,
    val categoryName: String,
    val categoryIcon: String,
    val date: Long,
    val note: String,
    val receiptImage: ByteArray? = null   // ← stored as BLOB
)
```
- `ByteArray?` maps to a **BLOB** column in SQLite (Binary Large Object). Room handles the conversion automatically.
- The field is **nullable** (`?`) because most expenses will not have a receipt.

**The ViewModel saves it:**
```kotlin
expenseDao.insertExpense(
    Expense(
        id = id,
        amount = doubleAmount,
        ...
        receiptImage = receiptImageBytes  // the compressed ByteArray
    )
)
```

**Split query approach (important design decision):**
The app uses a separate `ExpenseSummary` projection for lists and totals:
```kotlin
val allExpensesSummary: Flow<List<ExpenseSummary>> = expenseDao.getAllExpensesSummary()
```
- `ExpenseSummary` is a DTO (Data Transfer Object) that contains all fields *except* `receiptImage`.
- The image is only fetched when the user opens a specific expense detail screen:
```kotlin
fun loadFullExpense(id: String) {
    viewModelScope.launch {
        viewingExpense = expenseDao.getExpenseSummaryById(id)
        viewingReceiptImage = expenseDao.getExpenseImage(id) // ← separate query
    }
}
```
- **Why?** Loading the image blob for every row in a list would hit the 2MB CursorWindow limit almost instantly if multiple receipts exist. Fetching it only on demand is the correct pattern.

---

### 2.5 Displaying the Image

In `CreateExpenses.kt` (preview before saving):
```kotlin
if (receiptImageBytes != null) {
    val bitmap = remember(receiptImageBytes) {
        BitmapFactory.decodeByteArray(receiptImageBytes, 0, receiptImageBytes!!.size)
    }
    Image(
        bitmap = bitmap.asImageBitmap(),
        contentDescription = "Receipt preview",
        modifier = Modifier.height(180.dp).clip(RoundedCornerShape(12.dp)),
        contentScale = ContentScale.Crop
    )
}
```
- `remember(receiptImageBytes)` caches the decoded bitmap — it is only re-decoded if `receiptImageBytes` changes, preventing expensive work on every recompose.
- `asImageBitmap()` converts Android's `Bitmap` into a Compose-compatible `ImageBitmap`.

---

### Likely Lecturer Questions on Receipt Upload

**Q: Why store images in Room instead of Firebase Storage or the file system?**
A: For this prototype, local-only storage was chosen to keep things simple and offline-capable. The compressed byte array approach works within Room's limits after compression. A production app would typically upload to Firebase Storage and only store the download URL in Room.

**Q: What is a BLOB?**
A: Binary Large Object — a raw sequence of bytes stored in a database column. SQLite supports it natively. Room maps `ByteArray` to BLOB automatically.

**Q: What happens if the user denies the camera permission?**
A: The `cameraPermissionLauncher` callback receives `isGranted = false`. Currently the app logs an error (`Log.e`). A fully polished version would show a Snackbar or dialog explaining why the permission is needed.

**Q: Could a very large image still crash the app?**
A: The compression reduces any image to under 1024px on its longest side at 60% quality — typically well under 200KB. Room's 2MB CursorWindow limit is per query window (covering multiple rows), so the separate image-fetch query (one row, one blob) is the safe pattern used here.

**Q: What is `ContentResolver` and why is it used?**
A: `ContentResolver` is Android's standard API for accessing data from content providers — including the media store, file picker results, and camera URIs. It abstracts away the actual file system location, which can vary depending on Android version, manufacturer, and where the file lives (internal, SD card, cloud-backed).

**Q: What does `viewModelScope.launch` do?**
A: It launches a coroutine (an async task) that is tied to the ViewModel's lifecycle. If the ViewModel is destroyed (e.g. the user leaves the screen permanently), the coroutine is automatically cancelled. Database operations must run off the main thread — coroutines handle this cleanly without callbacks or threads.
