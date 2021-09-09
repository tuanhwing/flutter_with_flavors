# Example with flavors
Example app showing Flutter flavors configuration for Android and iOS.
I am going to build a sample app with three flavors: **develop, staging, production**.
Flutter flavors support is not very well documented in the [official docs](https://flutter.dev/docs/deployment/flavors) (yet) - they point to the following three articles as of writing
- Config Variables for your Flutter apps with [flutter_config](https://pub.dev/packages/flutter_config)
- [Flutter Launcher Icons](https://pub.dev/packages/flutter_launcher_icons)
- [Build flavors in Flutter(Android and iOS) with different Firease project per flavor](https://medium.com/@animeshjain/build-flavors-in-flutter-android-and-ios-with-different-firebase-projects-per-flavor-27c5c5dac10b)

***Running your app***:
This project has the following flavors:
- production: `flutter run --flavor production`
- staging: `flutter run --flavor staging`
- develop: `flutter run --flavor develop`

Android             |  iOS
:-------------------------:|:-------------------------:
![Screenshot](https://i.imgur.com/iiDKym3.png)  |  ![Screenshot](https://i.imgur.com/oqcjxkP.png)
# Why do you need flavors
Flavors are typically to build your app for different environments such as **develop** and **production**.
- **develop** version of the app might want to point to an API host at ```dev.api.myapp.com```
- **production** version of the app might want to point to ```api.myapp.com```

Instead of hardcoding these values into variables and building an app for every environment manually, the right approach is to use flavors, and supply these values as build time configurations.
# Getting Started

##### Step 0: Configuration to your `pubspec.yaml`:
```
dependencies:
    flutter_config: 2.0.0
    flutter_launcher_icons: 0.9.2
    firebase_core: 1.6.0
    firebase_messaging: 10.0.6
```

##### Step 1: Build flavors and config variables for your Flutter apps with [flutter_config](https://pub.dev/packages/flutter_config)
Save config for different environments in different files: `.env`, `.env.staging`, `.env.dev`
 - Create a new file `.env` in the root of your Flutter app:
    ```
    ENVIRONMENT=PRODUCTION
     ```
- Load all environment variables in `main.dart`
    ```
    import 'package:flutter_config/flutter_config.dart';

    void main() async {
      WidgetsFlutterBinding.ensureInitialized(); // Required by FlutterConfig
      await FlutterConfig.loadEnvVariables();
    
      runApp(MyApp());
    }
    ```

###### Android Setup
1. Manually apply a plugin to your app, from `android/app/build.gradle`:
    Below `apply from: "$flutterRoot/packages/flutter_tools/gradle/flutter.gradle"`, add the following lines:
    ```
    project.ext.envConfigFiles = [
        developdebug: ".env.dev",
        developrelease: ".env.dev",
        stagingdebug: ".env.staging",
        stagingrelease: ".env.staging",
        productiondebug: ".env",
        productionrelease: ".env",
    ]
    apply from: project(':flutter_config').projectDir.getPath() + "/dotenv.gradle"
    ```
    Add productFlavors look like:
    ```
    android {
        ...
        flavorDimensions "flavors"

        productFlavors {
            develop {
                dimension "flavors"
                applicationIdSuffix ".dev"
                manifestPlaceholders = [appName: "[DEV] example_with_flavors"]
            }
            staging {
                dimension "flavors"
                applicationIdSuffix ".stg"
                manifestPlaceholders = [appName: "[STG] example_with_flavors"]
            }
            production {
                dimension "flavors"
                applicationIdSuffix ""
                manifestPlaceholders = [appName: "example_with_flavors"]
            }
        }
    }
    ```
    To support different build variants. 
    ```
    defaultConfig {
        ...
        resValue "string", "build_config_package", "YOUR_PACKAGE_NAME_IN_ANDROIDMANIFEST.XML"
    }
    ```
    **NOTE** : Where `YOUR_PACKAGE_NAME_IN_ANDROIDMANIFEST.XML` should be replaced with your app's package name.
    
2. Also make sure `${appName}` value is assigned for `android:label` attribute in the manifest (`android/app/src/main/AndroidManifest.xml`)
    ```
    <application
        ...
        android:label="${appName}"
        ...
    ```
3. Build a release version
    When building your apk for release, the R8 code shrinker obfuscates the `BuildConfig` class which holds all the env variables and thus causes all the env variables to be null. To prevent this, the following has to be done:
-- Add file `android/app/proguard-rules.pro` to your app's project.
-- Add the below line to the newly created `proguard-rules.pro` file:

    ```
    -keep class com.yourcompany.app.BuildConfig { *; }
    ```
    where `com.yourcompany.app` should be replaced with your app's package name.
    
###### iOS Setup
1. Under `Runner/Flutter`. Add the following code to both `Debug.xcconfig` and `Release.xcconfig`
    ```
    #include? "tmp.xcconfig"
    ```
    ![Screenshot](https://i.imgur.com/xLDGFFV.png)
    It is also recommended to add this file to gitignore
    ```
    **/ios/Flutter/tmp.xcconfig
    ```
2. In the Xcode menu, go to `Product` > `Scheme` > `Edit Scheme`
    ![Screenshot](https://raw.githubusercontent.com/ByneappLLC/flutter_config/master/doc/pic2.png)
3. Under `Build` > `Pre-actions` you need to add 2 new run script actions with the following code:
    ![Screenshot](https://github.com/ByneappLLC/flutter_config/raw/master/doc/pic3.png)
    ```
    echo ".env" > ${SRCROOT}/.envfile
    ```
    ```
    ${SRCROOT}/.symlinks/plugins/flutter_config/ios/Classes/BuildXCConfig.rb ${SRCROOT}/ ${SRCROOT}/Flutter/tmp.xcconfig
    ```
4. Make sure you select `Runner` from the `Provide build settings from` dropdown
    Your finished scripts should look like this:
    ![Screenshot](https://github.com/ByneappLLC/flutter_config/raw/master/doc/pic5.png)
5. Creating new scheme for `develop` and `staging` environments:
    - In the Xcode menu, go to `Product` > `Scheme` > `Edit Scheme`
    - Click `Duplicate Scheme` on the bottom
    - Give it a proper name (`develop` , `staging`) on the top left.
    - In the Xcode menu, go to `Product` > `Scheme` > `Edit Schemes` > select your scheme (`develop`, `staging`) > Follow steps `3` and `4` again for each scheme and replace you env files
        ```
        echo ".env.staging" > ${SRCROOT}/.envfile   # replace .env.staging for your file
        ```
        ```
        echo ".env.dev" > ${SRCROOT}/.envfile   # replace .env.dev for your file
        ```
6. Rename `Runner` scheme to `production`
    - In the Xcode menu, go to `Product` > `Scheme` > `Manage Schemes...`
    - Select `Runner` and rename to `production`
        ![Screenshot](https://i.imgur.com/WQsUpMP.png)
7. Create build configurations...
    - Target project > `PROJECTS` > `Runner` > `Info` > `Configurations`
    - Create `develop` and `staging` configurations
        - Click `+` > Duplicate [`"Debug"`, `"Release"`, `"Profile"`] Configuration
        - Rename `Debug copy`, `Release copy`, `Profile copy` to:
            - `Debug-develop`, `Release-develop`, `Profile-develop`
            - `Debug-staging`, `Release-staging`, `Profile-staging`
    - Then rename `Debug`, `Release`, `Profile` to `Debug-production`, `Release-production`, `Profile-production`
    ![Screenshot](https://i.imgur.com/vrqBv1y.png)
8. Change the app name to be different for all schemes:
    - Target project > `TARGETS` > `Runner` > `Build Settings` > Click `+` > `Add User-Defined Setting` > Rename `NEW_SETTING` to `APP_DISPLAY_NAME` > Give proper name for all schemes...
    ![Screenshot](https://i.imgur.com/srYKw6l.png)
    - Update `Info.plist` to include `APP_DISPLAY_NAME` property
        ```
        <dict>
        ...
        <key>CFBundleDisplayName</key>
        <string>$(APP_DISPLAY_NAME)</string>
        ...
        </dict>
        ```
9. Change the app bundle identifier to be different for all schemes:
    - Target project > `TARGETS` > `Runner` > `Build Settings` > `Packing` > `Product Bundle Identifier` > Give proper bundle identifier for all schemes...
    ![Screenshot](https://i.imgur.com/AroHEPu.png)

###### Running your app
 - Setting `Build Flavor`, Android Studio > Click `main.dart` dropdown button > `Edit Configurations` > Set flavor name `develop`/`staging`/`production` at `Build flavor` > `Apply` > `OK` > Click `Run` button.
    ![Screenshot](https://i.imgur.com/KFvin2T.png)
 - Command
    ```
    flutter run --flavor develop
    flutter build apk --flavor staging
    ...
    ```

    
##### Step 2: Create Flutter Launcher Icons configuration file for all falvors
1. Copy launcher icons into `assets/launcher_icon` folder.
2. Config file is called `flutter_launcher_icons-<flavor>.yaml` by replacing ***<flavor>*** by the name of your desired flavor.
    ![Screenshot](https://i.imgur.com/IuruNad.png)
    For instance: **develop**
    ```
    flutter_icons:
      android: true
      ios: true
      image_path: "assets/launcher_icon/demo-icon-dev.png"
    ```
3. Run this command to create all launcher icons for all flavors:
    ```
    flutter pub run flutter_launcher_icons:main -f flutter_launcher_icons-${flavor_name}.yaml
    ```
4. ***(OPTIONAL)*** Create adaptive icons for Android:
    - Select `android` > `Flutter` > `Open Android module in Android Studio`
    - Select your format project: `Project`
    - `android` > `app` > `src` > Select ***<flavor_folder>*** then `Right click` > `New` > `Image Asset` > give image path at `Path` > `Next`.
    ![Screenshot](https://i.imgur.com/ot4MYlZ.png)
    - Select **flavor_name** at `Res Directory` > click `Finish`
    ![Screenshot](https://i.imgur.com/Ah5qLog.png)
    - Final it should be look like:
    ![Screenshot](https://i.imgur.com/tGt09JB.png)
5. Make sure iOS project is configured properly.
    - `Runner` > `Assets.xcassets` look like:
    ![Screenshot](https://i.imgur.com/ZYSAxaq.png)
    - Select **Project Target** > `TARGETS` > `Runner` > `Build Settings` > `Asset Catalog Complier - Options` > `Asset Catalog App Icon Set Name` > Correct flavor is mapping with App Icon.
    ![Screenshot](https://i.imgur.com/KgQNUwE.png)
6. Now, run your flavor and view app icon.

##### Step 3: Different Firease project per flavor
###### Create the app for iOS and Android in Firebase console and download `GoogleService-Info.plist` and `google-services.json` respectively:
![Screenshot](https://i.imgur.com/tKBX25o.png)

###### Android setup:
You can have multiple `google-services.json` files for different **build variants** by replacing `google-services.json` files in dedicated directories named for each variant under the app module root. Configuration could be organized like this:
![Screenshot](https://i.imgur.com/jIqkBIu.png)

###### iOS setup:
1. First, we will keep the `GoogleServices-Info.plist` files for each flavor in a separate folder like follows...
![Screenshot](https://i.imgur.com/SOVMOSD.png)
2. `GoogleServices-Info.plist` file is copied into the correct location, i.e inside the `Runner` directory. This can be accomplished by adding a new **Run script Build Phase** to the target...
![Alt Text](https://miro.medium.com/max/700/1*5tcidmX4DPIvskHpgtec0Q.gif)
The script I used is below:
    ```
    environment="develop"
    
    # Regex to extract the scheme name from the Build Configuration
    # We have named our Build Configurations as Debug-dev, Debug-prod etc.
    # Here, dev and prod are the scheme names. This kind of naming is required by Flutter for flavors to work.
    # We are using the $CONFIGURATION variable available in the XCode build environment to extract 
    # the environment (or flavor)
    # For eg.
    # If CONFIGURATION="Debug-prod", then environment will get set to "prod".
    if [[ $CONFIGURATION =~ -([^-]*)$ ]]; then
    environment=${BASH_REMATCH[1]}
    fi
    
    echo $environment
    
    # Name and path of the resource we're copying
    GOOGLESERVICE_INFO_PLIST=GoogleService-Info.plist
    GOOGLESERVICE_INFO_FILE=${PROJECT_DIR}/config/${environment}/${GOOGLESERVICE_INFO_PLIST}
    
    # Make sure GoogleService-Info.plist exists
    echo "Looking for ${GOOGLESERVICE_INFO_PLIST} in ${GOOGLESERVICE_INFO_FILE}"
    if [ ! -f $GOOGLESERVICE_INFO_FILE ]
    then
    echo "No GoogleService-Info.plist found. Please ensure it's in the proper directory."
    exit 1
    fi
    
    # Get a reference to the destination location for the GoogleService-Info.plist
    # This is the default location where Firebase init code expects to find GoogleServices-Info.plist file
    PLIST_DESTINATION=${BUILT_PRODUCTS_DIR}/${PRODUCT_NAME}.app
    echo "Will copy ${GOOGLESERVICE_INFO_PLIST} to final destination: ${PLIST_DESTINATION}"
    
    # Copy over the prod GoogleService-Info.plist for Release builds
    cp "${GOOGLESERVICE_INFO_FILE}" "${PLIST_DESTINATION}"
    
    ```

##### Done! Now we can run:
`flutter run --flavor develop` or `flutter run --flavor staging`
And the iOS/Android will install separate apps which connect to separate Firebase!