---
layout: post
author: Shahid Raza
title: Stop Repeating Yourself - Master Convention Plugins in Gradle
tags: Android Gradle Build System
math: false
---
Recently I got to know about convention plugins in gradle, it eliminates duplicated build logic across multi-module projects. This guide shows you how to enforce consistency, improve performance, and scale your Gradle builds with an included build setup.

<!--more--> 

**Convention Plugins in Gradle: Streamlining Multi-Module Builds**

Gradle is the backbone of many modern JVM, Android, and Kotlin projects. As projects grow—especially multi-module ones—build scripts balloon with duplicated configuration for dependencies, compiler options, Android settings, testing, linting, and publishing. This leads to maintenance nightmares, inconsistencies, and slower builds.

**Convention plugins** solve this elegantly. They are a recommended Gradle pattern for sharing and enforcing build conventions across projects without the pitfalls of `subprojects {}` / `allprojects {}` blocks or duplicated script plugins.

### What Are Convention Plugins?

A **convention plugin** is typically a *precompiled script plugin* (or a binary plugin) that applies and configures core/community plugins with your team's defaults (for example -  Java/Kotlin version, Android SDK settings, common dependencies). 

- They centralize logic in one place.
- They are applied like any other plugin (`id("my.convention.android-library")`).
- They promote **project isolation**, Configuration Cache compatibility, and better performance compared to cross-project configuration.

Gradle docs emphasize them as the preferred way to share build logic.

### Why Use Them?

- **DRY (Don't Repeat Yourself)**: Eliminate copy-paste across dozens of modules.
- **Consistency**: Enforce standards (for example -  same `minSdk`, compiler flags, test runners).
- **Maintainability**: Update once, affects everywhere.
- **Performance**: Better than root-project references or script plugins; supports modern Gradle features.
- **Scalability**: Ideal for large monorepos or Android apps with many feature modules.
- **Extensibility**: Add custom extensions for module-specific tweaks.

Common pain points they solve: version management (pair with Version Catalogs), boilerplate Android configs, and third-party plugin setup (for example -  Kotlin, Compose, KSP, Detekt).

### Two Main Ways to Set Them Up

1. **`buildSrc`** (simplest, great for small/medium projects): Automatically available. Limited to one build.
2. **Included build** (recommended for larger setups, for example -  `build-logic/`): More scalable, composite build.

We'll focus on the included build approach, as it's the current best practice.

### Step-by-Step: Creating a Convention Plugin (Android Example)

Assume a multi-module Android project with a Version Catalog (`gradle/libs.versions.toml`).

#### 1. Create the `build-logic` Included Build

Project structure:
```
.
├── build-logic/
│   ├── settings.gradle.kts
│   ├── build.gradle.kts
│   └── convention/          # or src/main/kotlin for precompiled
│       └── src/main/kotlin/
├── gradle/libs.versions.toml
├── settings.gradle.kts     # root
└── app/, library1/, etc.
```

**`build-logic/settings.gradle.kts`**:
```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}
rootProject.name = "build-logic"
```

**`build-logic/build.gradle.kts`** (for a binary plugin):
```kotlin
plugins {
    `java-gradle-plugin`
    alias(libs.plugins.kotlin.jvm)
}

java {
    toolchain { languageVersion.set(JavaLanguageVersion.of(17)) }
}

dependencies {
    compileOnly(libs.android.gradlePlugin)  // or whatever plugins you need
    implementation(gradleKotlinDsl())
}

gradlePlugin {
    plugins {
        register("androidLibrary") {
            id = "com.example.convention.android.library"
            implementationClass = "com.example.convention.AndroidLibraryConventionPlugin"
        }
        // Add more: androidApplication, jvmLibrary, etc.
    }
}
```

#### 2. Implement the Plugin

**`build-logic/convention/src/main/kotlin/com/example/convention/AndroidLibraryConventionPlugin.kt`**:
```kotlin
import com.android.build.api.dsl.LibraryExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.*

class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        pluginManager.apply("com.android.library")

        extensions.configure<LibraryExtension> {
            compileSdk = 36  // from catalog or constant

            defaultConfig {
                minSdk = 24
                testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
                consumerProguardFiles("consumer-rules.pro")
            }

            buildTypes {
                release {
                    isMinifyEnabled = false
                    proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
                }
            }

            compileOptions {
                sourceCompatibility = JavaVersion.VERSION_11
                targetCompatibility = JavaVersion.VERSION_11
            }
            // Add Kotlin, Compose, namespace handling, etc.
        }

        // Common dependencies via Version Catalog
        val libs = extensions.getByType<VersionCatalogsExtension>().named("libs")
        dependencies {
            add("implementation", libs.findLibrary("androidx-core-ktx").get())
            // ... more commons
        }
    }
}
```

#### 3. Wire It Up in Root `settings.gradle.kts`

```kotlin
pluginManagement {
    includeBuild("build-logic")
    repositories { ... }
}

// In root build.gradle.kts, declare plugins with apply false if needed
```

#### 4. Use in Modules

**`library1/build.gradle.kts`**:
```kotlin
plugins {
    id("com.example.convention.android.library")
}

android {
    namespace = "com.example.library1"
    // Override specifics if needed
}
```

That's it—modules become tiny and consistent.

### Advanced Tips

- **Extensions for Configurability**: Expose an extension (for example -  `myconvention { enableCompose() }`) so modules can opt into extras.
- **Precompiled Script Plugins**: Simpler alternative—place `.gradle.kts` files in `build-logic/convention/src/main/kotlin/` (filename becomes plugin ID).
- **Dependencies**: Use `compileOnly` for plugins you apply inside; `implementation` for helpers.
- **Version Catalogs**: Share versions everywhere.
- **Testing & Publishing**: For team-wide use, publish your convention plugins to a repository.
- **Performance**: Avoid heavy logic; test with Configuration Cache (`--configuration-cache`).
- **Binary vs. Precompiled**: Binary (`.kt` classes) offers more power (custom tasks, extensions); precompiled is quicker to start.

### Common Patterns

- **Java/Kotlin Library Convention**: Toolchains, repositories, testing (JUnit 5), publishing.
- **Android App/Library**: AGP config, flavors, signing (via properties), lint, ProGuard.
- **Compose/Hilt/KSP**: Conditional application.
- **Quality**: Detekt, KtLint, Jacoco integration.

### Potential Pitfalls

- Setup complexity (especially understanding included builds).
- IDE sync issues during initial setup—clean/re-sync often.
- Plugin application order matters.
- For very large projects, consider publishing conventions separately for faster builds.

### Conclusion

Convention plugins transform chaotic build scripts into clean, declarative, and maintainable code. They align with Gradle's vision for scalable builds and pay dividends as your project grows.

Start small: Extract one common configuration (for example -  Android library basics) into a plugin, then expand. Your future self (and teammates) will thank you.

**Further Reading**:
- Official Gradle Docs on Convention Plugins
- Android/NowInAndroid examples
- Community repos like krogerco/gradle-convention-plugins

What convention would you extract first in your project? Hit me up on LinkedIn!
