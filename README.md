# Build Environment Setup for Kotlin Multiplatform

## Introduction
This tutorial will guide you through setting up different build environments for a Kotlin Multiplatform project. We will create separate modules for each environment and configure them appropriately. We will select the module to use by passing a project flag (-P) when building with Gradle.

## Build Command
```shell
./gradlew build -Penvironment=dev
```

### File: `composeApp/build.gradle.kts`
```kotlin
val environment: String by project
```

### File: `gradle.properties`
```
environment=dev
```

## Shared Setup

### Setting Up the Modules
Create the following structure for the environment-specific configurations:

```
config/dev
    src
        androidMain
            res
                values
                    strings.xml
        commonMain
            kotlin
                Config.kt
        build.gradle.kts
```

#### File: `config/dev/src/androidMain/res/values/strings.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">Environment (DEV)</string>
</resources>
```
**Note:** Remove the `app_name` entry in the `composeApp` module.

#### File: `config/dev/src/commonMain/kotlin/Config.kt`
```kotlin
object Config {
    const val ENVIRONMENT: String = "development"
    const val isProduction: Boolean = false
}
```

#### File: `config/dev/build.gradle.kts`
```kotlin
plugins {
   alias(libs.plugins.kotlinMultiplatform)
   alias(libs.plugins.androidLibrary)
}

kotlin {
   androidTarget {
       compilations.all {
           kotlinOptions {
               jvmTarget = "17"
           }
       }
   }

   listOf(
       iosX64(),
       iosArm64(),
       iosSimulatorArm64()
   ).forEach {
       it.binaries.framework {
           baseName = "environment"
           isStatic = true
       }
   }

   sourceSets {
       commonMain.dependencies {
           
       }
       commonTest.dependencies {
           
       }
   }
}

android {
    namespace = "com.rakangsoftware.environment"
    compileSdk = libs.versions.android.compileSdk.get().toInt()
    
    defaultConfig {
        minSdk = libs.versions.android.minSdk.get().toInt()
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}
```
**Note:** Repeat these steps for all other environments.

### Adding Modules to Project
#### File: `settings.gradle.kts`
Add the environment modules to the project.
```kotlin
include(":config:dev", ":config:prod")
```

### Using the Selected Environment Module
#### File: `composeApp/build.gradle.kts`
```kotlin
kotlin {
    // ...
    
    sourceSets {
        // ...

        commonMain.dependencies {
            // Environment Configuration
            implementation(project(":config:$environment"))

            // ...
        }

    }
}
```

## Android Specific Setup

### Changing the Package ID
```kotlin
val suffix = when (environment) {
   "prod" -> ""
   else -> ".$environment"
}
println("The current build environment is set to '$environment' with suffix set to '$suffix'.")
```

### Setting the `applicationIdSuffix`
```kotlin
android {
   // ...

   defaultConfig {
       applicationId = "com.rakangsoftware.environment"
       applicationIdSuffix = suffix
       minSdk = libs.versions.android.minSdk.get().toInt()
       targetSdk = libs.versions.android.targetSdk.get().toInt()
       versionCode = 1
       versionName = "0.0.1"
   }
}
```

### Gradle Task to Run on Device
```kotlin
tasks.register("runDebug", Exec::class) {
   dependsOn("clean", "installDebug")

   doFirst {
       exec {
           commandLine("adb", "shell", "am", "force-stop", "com.rakangsoftware.environment$suffix")
       }
   }

   commandLine(
       "adb", "shell", "am", "start", "-n",
       "com.rakangsoftware.environment$suffix/com.rakangsoftware.environment.MainActivity"
   )
}
```

### Run Configuration
In Android Studio, configure the run settings:
1. Go to `Run > Edit Configurations`.
2. Press the `+` button in the top left corner and select `Gradle`.
3. Name it: `Android Dev`.
4. In the `Run` field, select `composeApp:runDebug` and add the `-Penvironment=dev` flag.
5. Optionally, check the `Store as project file` box.

Repeat these steps for all environments, adjusting the `-Penvironment` flag accordingly.

## iOS Specific Setup

### Configuration Files
In Xcode:
1. Right-click on the `Configuration` folder and select `New File...`.
2. Choose `Configuration Setting File` and add the following content:
    ```shell
    TEAM_ID=
    BUNDLE_ID=com.rakangsoftware.environment.dev
    APP_NAME=Environment
    APPLICATION_DISPLAY_NAME=Environment (DEV)
    ENVIRONMENT=dev
    KOTLIN_FRAMEWORK_BUILD_TYPE=debug
    ```
3. Repeat for each environment, changing `KOTLIN_FRAMEWORK_BUILD_TYPE` to `release` for production.

### Setting the Product and Bundle Names
1. In Xcode, select the root folder (`iosApp`) and the target (`iosApp`).
2. Under the `Build Settings` tab, locate `Product Name` and enter: `${APP_NAME}`.
3. In the `info.plist` file, find `Bundle Name` and change the value to: `$(APPLICATION_DISPLAY_NAME)`.

### Configurations
1. In Xcode, select the project (`iosApp`) and go to the `Info` tab.
2. Under `Configurations`, create new configurations:
    - Duplicate "Debug" Configuration and name it `Dev Debug`.
    - Duplicate "Release" Configuration and name it `Dev Release`.
    - Assign `DEV` to these configurations.
    - Repeat for `PROD`, assigning `PROD`.

### Build Schemes
1. Click `Manage Schemes...`.
2. Create a new scheme, name it `iOS Dev`, and set the build configurations:
    - `Run`, `Test`, `Analyze`: `Dev Debug`
    - `Profile`, `Archive`: `Dev Release`
3. Repeat for other environments.

### Environment Flag
1. Select the target `iosApp` and go to the `Build Phases` tab.
2. In the `Compile Kotlin Framework` section, find the line:
    ```shell
    ./gradlew :composeApp:embedAndSignAppleFrameworkForXcode
    ```
3. Add the flag:
    ```shell
    ./gradlew :composeApp:embedAndSignAppleFrameworkForXcode -Penvironment=$ENVIRONMENT
    ```

## Final Step: Run Configuration for iOS in Android Studio
1. Go to `File > Invalidate Caches...`, check all boxes, and press `Invalidate and Restart`.
2. Open `Edit Configurations...` and configure the run settings:
    - Select `iOS Application` and rename the default configuration `iosApp` to `iOS Dev Debug`.
    - Select the scheme `iOS Dev` and the configuration `Dev Debug`.
3. Repeat for each environment.
