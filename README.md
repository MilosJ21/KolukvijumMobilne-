# Kolokvijum2 – Android Aplikacija

## Struktura projekta

```
Kolokvijum2/
├── app/
│   ├── src/
│   │   └── main/
│   │       ├── java/com/example/kolokvijum2/
│   │       │   ├── MainActivity.kt          ← glavna aktivnost (senzori, kamera, UI logika)
│   │       │   ├── Continent.kt             ← data klasa za model kontinenta
│   │       │   ├── ApiService.kt            ← Retrofit interfejs
│   │       │   └── RetrofitClient.kt        ← Retrofit singleton
│   │       ├── res/
│   │       │   ├── layout/
│   │       │   │   └── activity_main.xml    ← layout sa checkboxovima, imagebutton, textview
│   │       │   └── xml/
│   │       │       └── file_paths.xml       ← FileProvider putanje za kameru
│   │       └── AndroidManifest.xml
│   └── build.gradle                         ← dependencies (Retrofit, Room, itd.)
```

---

## 1. `build.gradle` (app nivo) – Dependencies

> Dodaj u `dependencies {}` blok u `app/build.gradle`

```kotlin
dependencies {
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")
    implementation("androidx.room:room-runtime:2.6.1")
    kapt("androidx.room:room-compiler:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    // ostale standardne dependencies ostaju kako jesu
}
```

> Takođe u `plugins {}` bloku na vrhu `build.gradle` treba da stoji:
```kotlin
plugins {
    id("kotlin-kapt")
}
```

---

## 2. `AndroidManifest.xml` – Dozvole i FileProvider

> Unutar `<manifest>` taga, pre `<application>` taga, dodaj dozvole:

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-feature android:name="android.hardware.camera" android:required="false" />
```

> Unutar `<application>` taga (pored `<activity>`), dodaj FileProvider:

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

---

## 3. `res/xml/file_paths.xml` – FileProvider putanje

> Napravi folder `res/xml/` ako ne postoji, pa napravi fajl `file_paths.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <cache-path
        name="camera_images"
        path="images/" />
</paths>
```

---

## 4. `res/layout/activity_main.xml` – Layout

> Zameni kompletan sadržaj `activity_main.xml` sledećim:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <CheckBox
        android:id="@+id/checkboxFetch"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Preuzmi kontinente" />

    <CheckBox
        android:id="@+id/checkboxCountries"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Prikaži broj država 3. kontinenta" />

    <ImageButton
        android:id="@+id/imageButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:src="@android:drawable/ic_menu_camera"
        android:contentDescription="Otvori kameru"
        android:layout_marginTop="16dp"/>

    <TextView
        android:id="@+id/textViewProximity"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Proximity senzor: čekanje..."
        android:layout_marginTop="16dp"
        android:textSize="16sp"/>

    <TextView
        android:id="@+id/textViewInfo"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Info:"
        android:layout_marginTop="16dp"
        android:textSize="16sp"/>

</LinearLayout>
```

---

## 5. `Continent.kt` – Data klasa i Room entitet

> Napravi novi Kotlin fajl `Continent.kt` u paketu `com/example/kolokvijum2/`

```kotlin
package com.example.kolokvijum2

import androidx.room.Entity
import androidx.room.PrimaryKey
import com.google.gson.annotations.SerializedName

// Ova klasa je i Room entitet (baza) i Retrofit model (API)
@Entity(tableName = "continents")
data class Continent(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,

    @SerializedName("name")
    val name: String,

    @SerializedName("population")
    val population: Long,

    @SerializedName("countriesCount")   // naziv polja sa API-ja
    val countriesCount: Int
)
```

> **Napomena:** Proveri na `beeceptor.com/mock-server/dummy-json` kako se zovu polja u JSON odgovoru
> i ako se razlikuju, prilagodi `@SerializedName("...")` vrednosti.

---

## 6. `ContinentDao.kt` – Room DAO

> Napravi novi fajl `ContinentDao.kt`:

```kotlin
package com.example.kolokvijum2

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query

@Dao
interface ContinentDao {

    // Umetanje jednog kontinenta
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(continent: Continent)

    // Dohvati sve kontinente iz baze
    @Query("SELECT * FROM continents")
    suspend fun getAll(): List<Continent>

    // Dohvati treći kontinent (LIMIT 1 OFFSET 2 = treći red)
    @Query("SELECT * FROM continents LIMIT 1 OFFSET 2")
    suspend fun getThird(): Continent?
}
```

---

## 7. `AppDatabase.kt` – Room baza

> Napravi novi fajl `AppDatabase.kt`:

```kotlin
package com.example.kolokvijum2

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

@Database(entities = [Continent::class], version = 1)
abstract class AppDatabase : RoomDatabase() {

    abstract fun continentDao(): ContinentDao

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getDatabase(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "continent_database"
                ).build()
                INSTANCE = instance
                instance
            }
        }
    }
}
```

---

## 8. `ApiService.kt` – Retrofit interfejs

> Napravi novi fajl `ApiService.kt`:

```kotlin
package com.example.kolokvijum2

import retrofit2.http.GET

interface ApiService {

    // GET zahtev – endpoint /continents na beeceptor mock serveru
    @GET("continents")
    suspend fun getContinents(): List<Continent>
}
```

---

## 9. `RetrofitClient.kt` – Retrofit singleton

> Napravi novi fajl `RetrofitClient.kt`:

```kotlin
package com.example.kolokvijum2

import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

object RetrofitClient {

    // Zameni URL sa tačnim URL-om tvog mock servera na beeceptor-u
    private const val BASE_URL = "https://beeceptor.com/mock-server/dummy-json/"

    val api: ApiService by lazy {
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(ApiService::class.java)
    }
}
```

---

## 10. `MainActivity.kt` – Glavna aktivnost

> Ovo je centralna klasa koja sve spaja. Zameni kompletan sadržaj `MainActivity.kt`:

```kotlin
package com.example.kolokvijum2

import android.Manifest
import android.content.Context
import android.content.Intent
import android.content.pm.PackageManager
import android.hardware.Sensor
import android.hardware.SensorEvent
import android.hardware.SensorEventListener
import android.hardware.SensorManager
import android.location.Location
import android.net.Uri
import android.os.Bundle
import android.os.Environment
import android.provider.MediaStore
import android.widget.CheckBox
import android.widget.ImageButton
import android.widget.TextView
import android.widget.Toast
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.core.app.ActivityCompat
import androidx.core.content.FileProvider
import androidx.lifecycle.lifecycleScope
import com.google.android.gms.location.FusedLocationProviderClient
import com.google.android.gms.location.LocationServices
import kotlinx.coroutines.launch
import java.io.File

class MainActivity : AppCompatActivity(), SensorEventListener {

    // ---------- UI elementi ----------
    private lateinit var checkboxFetch: CheckBox
    private lateinit var checkboxCountries: CheckBox
    private lateinit var imageButton: ImageButton
    private lateinit var textViewProximity: TextView
    private lateinit var textViewInfo: TextView

    // ---------- Senzori ----------
    private lateinit var sensorManager: SensorManager
    private var proximitySensor: Sensor? = null
    private var locationSensor: Sensor? = null   // stepCounter ili linearAcceleration (nema pravi location senzor)

    // ---------- Lokacija ----------
    private lateinit var fusedLocationClient: FusedLocationProviderClient

    // ---------- Baza ----------
    private lateinit var db: AppDatabase

    // ---------- Kamera ----------
    private var photoUri: Uri? = null
    private var photoFile: File? = null

    // Prag za proximity senzor (u cm) – ispod ovog = blizu
    private val PROXIMITY_THRESHOLD = 5.0f

    // ---------- Launcher za kameru ----------
    private val cameraLauncher = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if (result.resultCode == RESULT_OK) {
            val path = photoFile?.absolutePath ?: "nepoznato"
            Toast.makeText(this, "Slika sačuvana: $path", Toast.LENGTH_LONG).show()
        }
    }

    // ---------- Launcher za lokacijske dozvole ----------
    private val locationPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        if (permissions[Manifest.permission.ACCESS_FINE_LOCATION] == true) {
            getLocation()
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // Povezi UI
        checkboxFetch = findViewById(R.id.checkboxFetch)
        checkboxCountries = findViewById(R.id.checkboxCountries)
        imageButton = findViewById(R.id.imageButton)
        textViewProximity = findViewById(R.id.textViewProximity)
        textViewInfo = findViewById(R.id.textViewInfo)

        // Inicijalizuj bazu
        db = AppDatabase.getDatabase(this)

        // Inicijalizuj senzore
        sensorManager = getSystemService(Context.SENSOR_SERVICE) as SensorManager
        proximitySensor = sensorManager.getDefaultSensor(Sensor.TYPE_PROXIMITY)

        // Inicijalizuj lokaciju
        fusedLocationClient = LocationServices.getFusedLocationProviderClient(this)

        // ---------- ImageButton – otvori kameru ----------
        imageButton.setOnClickListener {
            openCamera()
        }

        // ---------- Checkbox 1 – Preuzmi kontinente ----------
        checkboxFetch.setOnCheckedChangeListener { _, isChecked ->
            if (isChecked) {
                fetchAndSaveContinents()
            }
        }

        // ---------- Checkbox 2 – Prikaži info ----------
        checkboxCountries.setOnCheckedChangeListener { _, isChecked ->
            if (isChecked) {
                // Čekirano: prikaži broj država 3. kontinenta iz baze
                showThirdContinentCountries()
            } else {
                // Odčekirano: prikaži lokaciju iz senzora
                showLocation()
            }
        }
    }

    // =====================================================
    // KAMERA
    // =====================================================

    private fun openCamera() {
        // Napravi folder u cache direktorijumu
        val imageDir = File(cacheDir, "images")
        if (!imageDir.exists()) imageDir.mkdirs()

        // Napravi prazan fajl za sliku
        photoFile = File(imageDir, "photo_${System.currentTimeMillis()}.jpg")
        photoFile!!.createNewFile()

        // Napravi URI koristeći FileProvider (ne direktno file://)
        photoUri = FileProvider.getUriForFile(
            this,
            "${packageName}.fileprovider",
            photoFile!!
        )

        // Pokreni kameru sa tim URI-jem
        val intent = Intent(MediaStore.ACTION_IMAGE_CAPTURE).apply {
            putExtra(MediaStore.EXTRA_OUTPUT, photoUri)
        }
        cameraLauncher.launch(intent)
    }

    // =====================================================
    // PROXIMITY SENZOR
    // =====================================================

    override fun onResume() {
        super.onResume()
        // Registruj proximity senzor kada je aktivnost vidljiva
        proximitySensor?.let {
            sensorManager.registerListener(this, it, SensorManager.SENSOR_DELAY_NORMAL)
        }
    }

    override fun onPause() {
        super.onPause()
        // Odregistruj senzor kada aktivnost nije vidljiva (štedi bateriju)
        sensorManager.unregisterListener(this)
    }

    override fun onSensorChanged(event: SensorEvent?) {
        if (event == null) return

        when (event.sensor.type) {
            Sensor.TYPE_PROXIMITY -> {
                val distance = event.values[0]
                textViewProximity.text = "Proximity senzor: $distance cm"

                // Ako je vrednost ispod praga – pošalji Toast "Blizu!"
                if (distance < PROXIMITY_THRESHOLD) {
                    Toast.makeText(this, "Blizu!", Toast.LENGTH_SHORT).show()
                }
            }
        }
    }

    override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {
        // Nije potrebno implementirati za ovaj zadatak
    }

    // =====================================================
    // RETROFIT – Preuzimanje i čuvanje kontinenata
    // =====================================================

    private fun fetchAndSaveContinents() {
        lifecycleScope.launch {
            try {
                // GET zahtev ka API-ju
                val continents = RetrofitClient.api.getContinents()

                // Sačuvaj samo one čija populacija prelazi 10,000
                var savedCount = 0
                for (continent in continents) {
                    if (continent.population > 10_000) {
                        db.continentDao().insert(continent)
                        savedCount++
                    }
                }

                Toast.makeText(
                    this@MainActivity,
                    "Sačuvano $savedCount kontinenata (populacija > 10,000)",
                    Toast.LENGTH_LONG
                ).show()

            } catch (e: Exception) {
                Toast.makeText(
                    this@MainActivity,
                    "Greška: ${e.message}",
                    Toast.LENGTH_LONG
                ).show()
            }
        }
    }

    // =====================================================
    // ROOM – Prikaz broja država trećeg kontinenta
    // =====================================================

    private fun showThirdContinentCountries() {
        lifecycleScope.launch {
            val third = db.continentDao().getThird()
            if (third != null) {
                textViewInfo.text = "Broj država 3. kontinenta (${third.name}): ${third.countriesCount}"
            } else {
                textViewInfo.text = "Nema podataka u bazi. Prvo čekiraj 'Preuzmi kontinente'."
            }
        }
    }

    // =====================================================
    // LOKACIJA – Prikaz kada se odčekira checkbox 2
    // =====================================================

    private fun showLocation() {
        // Proveri dozvole
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION)
            != PackageManager.PERMISSION_GRANTED
        ) {
            locationPermissionLauncher.launch(
                arrayOf(
                    Manifest.permission.ACCESS_FINE_LOCATION,
                    Manifest.permission.ACCESS_COARSE_LOCATION
                )
            )
            return
        }
        getLocation()
    }

    private fun getLocation() {
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION)
            != PackageManager.PERMISSION_GRANTED
        ) return

        fusedLocationClient.lastLocation.addOnSuccessListener { location: Location? ->
            if (location != null) {
                textViewInfo.text = "Lokacija: ${location.latitude}, ${location.longitude}"
            } else {
                textViewInfo.text = "Lokacija nije dostupna."
            }
        }
    }
}
```

---

## Rezime toka aplikacije

| Akcija | Šta se dešava |
|---|---|
| App se pokrene | Registruje se proximity senzor |
| Senzor detektuje vrednost < 5 cm | Toast "Blizu!" + vrednost se ispisuje u `textViewProximity` |
| Klik na ImageButton | Otvara se kamera, slika se čuva u `cache/images/`, putanja se prikazuje u Toast-u |
| Čekiranje Checkbox 1 | Retrofit GET poziv → čuvaju se kontinenti sa populacijom > 10.000 u Room bazi |
| Čekiranje Checkbox 2 | Iz baze se čita 3. kontinent, broj država se ispisuje u `textViewInfo` |
| Odčekiranje Checkbox 2 | Traži se poslednja poznata GPS lokacija, ispisuje se u `textViewInfo` |

---

## Važne napomene

1. **Beeceptor URL** – Otvori `beeceptor.com/mock-server/dummy-json`, pronađi endpoint za kontinente i proveri tačna naziva polja u JSON-u. Prilagodi `@SerializedName` u `Continent.kt` i `@GET("...")` u `ApiService.kt`.
2. **`countriesCount` polje** – Ako API vraća npr. `"countries"` umesto `"countriesCount"`, promeni `@SerializedName("countries")`.
3. **FusedLocation** – Za lokaciju je potrebna dependency `implementation("com.google.android.gms:play-services-location:21.0.1")` u `build.gradle`.
4. **Toast za proximity** – Ako senzor stalno šalje iste vrednosti, Toast se može pojavljivati često. Može se dodati debounce flag ako smeta.
