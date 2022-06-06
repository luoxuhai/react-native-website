---
id: pillars-fabric-components
title: Fabric Components
---

import Tabs from '@theme/Tabs'; import TabItem from '@theme/TabItem'; import constants from '@site/core/TabsConstants';

A Fabric Component is a UI component which is rendered on the screen using the [Fabric Renderer](https://reactnative.dev/architecture/fabric-renderer).

Using Fabric Components instead of Native Components allows us to reap all the [benefit](./why) coming from the **New Architecture**. Specifically, we are able to leverage JSI to efficiently connect the Native UI code and the JavaScript one, and all the other features provided by the new renderer.

A Fabric Component is created starting from a **JS specification**. This, with the help of a bit of [**Codegen**](./pillars-codegen), will create some C++ code, shared among all the RN's platforms. Then, after filling in the missing **native code** with Component specific logic, the Component can be **integrated** in the app.

The following section will guide you through the creation of a Fabric Component, step-by-step.

:::caution
Fabric Components only works with the **New Architecture** enabled.
To migrate to the **New Architecture**, follow the [Migration guide](../new-architecture-intro)
:::

## How to Create a Fabric Components

To create a Fabric Component, we have to follow these steps:

- Define a set of JS specification.
- Configure the Component so that it can be consumed by an app.
- Write the native code required to make it works.

Once these steps are done, the Component is ready to be consumend by an app. Therefore, the guide shows how to add it to an app, leveraging the _autolinking_, and how to reference it from the JavaScript code.

### Folder Setup

The easiest way to create a Component is as a separate module we will then import as a dependency for our apps. This has several benefits, among which it pushes for code reuse, it keeps the components decoupled from the rest of the apps and it makes it easier to leverage some automation already in place.

For this guide, we are going to create a Fabric Component that centers some text on the screen.

Let's create a new folder at the same level of our app and let's call it `RTNCenteredText`.

In this folder, we are going to create three subfolders: `js`, `ios` and `android`.

The final result should look like this:

<figure>
  <img width="500" alt="Folder Structure for a Fabric Component" src="/docs/assets/NewArchitecture/AppFolderStructure.png"/>
  <figcaption>Initial folder structure for a Fabric Component.</figcaption>
</figure>

### JavaScript Specification

In the **New Architecture**, we decided to have a single source of truth for Components API and we decided that it has to be a typed dialect of JavaScript (either [Flow](https://flow.org/) or [TypeScript](https://www.typescriptlang.org/)). We need a typed dialect because the **Codegen** has to generate some code in C++, Objective-C++ and Java which are all strongly typed languages: therefore, it needs information on types.

Another important aspect of the JavaScript specification is the file name. A Component filename has to end with the `NativeComponent.js` (or `jsx`, `ts`, `tsx`) suffix. The **Codegen** process will look up for files ending with that suffix to generate the required code.

The following are the specification of our `RTNCenteredText` Component in both Flow and TypeScript.

<Tabs groupId="fabric-component-specs" defaultValue={constants.defaultJavaScriptSpecLanguages} values={constants.javaScriptSpecLanguages}>
<TabItem value="flow">

```typescript
// @flow strict-local

import type {ViewProps} from 'react-native/Libraries/Components/View/ViewPropTypes';
import type {HostComponent} from 'react-native';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';

type NativeProps = $ReadOnly<{|
  ...ViewProps,
  text: ?string,
  // add other props here
|}>;

export default (codegenNativeComponent<NativeProps>(
   'RTNCenteredText',
): HostComponent<NativeProps>);
```

</TabItem>
<TabItem value="typescript">

```typescript
import type { ViewProps } from 'ViewPropTypes';
import type { HostComponent } from 'react-native';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';

export interface NativeProps extends ViewProps {
  ...ViewProps,
  text: ?string,
  // add other props here
}

export default codegenNativeComponent<NativeProps>(
  'RTNCenteredText'
) as HostComponent<NativeProps>;
```

</TabItem>
</Tabs>

Let's break these down a little.

At the beginning of the spec files, there are the imports. Here, we can import what we need from React Native and other dependencies if needed. The most important imports, that are present in all the Fabric Components, are:

- The `HostComponent`, which represents what we have to export;
- The `codegenNativeComponent` function, which is responsible to actually register the Component in the JS runtime.

The second section of the files contains the **props** of the Component. Props are Component-specific information we want to be able to customize from the outside. In this case, we want to control the `text` the Component will render.

Finally, we invoke the `codegenNativeComponent` generic function, passing the name we want to use for our Component. The returned value is then exported by the JavaScript file in order to be used by the app.

### Component Configuration

The second element we need to properly develop the Component is a bit of configuration, that will help you setting up:

- the Codegen
- the files to link the Component in the app.

Some of these configuration are shared between iOS and Android, while the others are platform specific.

#### Shared

The shared bit is a `package.json` file that will be used by yarn to install your Component.
The file shall live at the root of your Component, side-by-side to the folders created at the [folder Setup](#folder-setup) step.

```json
{
  "name": "rnt-centered-text",
  "version": "0.0.1",
  "description": "Showcase a Fabric Component with a centered text",
  "react-native": "js/index",
  "source": "js/index",
  "files": [
    "js",
    "android",
    "ios",
    "rnt-centered-text.podspec",
    "!android/build",
    "!ios/build",
    "!**/__tests__",
    "!**/__fixtures__",
    "!**/__mocks__"
  ],
  "keywords": ["react-native", "ios", "android"],
  "repository": "https://github.com/<your_github_handle>/rnt-centered-text",
  "author": "<Your Name> <your_email@your_provider.com> (https://github.com/<your_github_handle>)",
  "license": "MIT",
  "bugs": {
    "url": "https://github.com/<your_github_handle>/rnt-centered-text/issues"
  },
  "homepage": "https://github.com/<your_github_handle>/rnt-centered-text#readme",
  "devDependencies": {},
  "peerDependencies": {
    "react": "*",
    "react-native": "*"
  },
  "codegenConfig": {
    "name": "RNTCenteredTextSpecs",
    "type": "all",
    "jsSrcsDir": "js",
    "android": {
      "javaPackageName": "com.RTNCenteredText"
    }
  }
}
```

Let's break it down.

The upper part of the file contains some descriptive information like the name of the Component, its version and what composes it.
Make sure to update the various placeholders which are wrapped in `<>`, so replace all the occurrences of the `<your_github_handle>`, `<Your Name>`, and `<your_email@your_provider.com>` tokens.

Then we have the dependencies for this package, specifically we need `react` and `react-native`, but you can add all the other dependencies you may have.

Finally, the last important bit is the `codegenConfig` tag. This will be used **In Development** to automatically generate the code. It will be useful to see what is generated so that we can refer to the proper types. This object contains three keys:

- `name`: The name of the library we are going to generate. By convention, we add the `Specs` suffix.
- `type`: You can choose between `components` and `all`. Our suggestion is to leave `all` because it will push for extensibility in case you'll want to add also a TurboModule to this Component.
- `jsSrcsDir`: the relative path to access the `js` specification that will be parsed by the Codegen.
- `android`: an object that contains android-specific settings.
  - `javaPackageName`: the name of the package that must be used to generate the code.

#### iOS: Create the `podspec` file

Now it's time to configure the native Component so that we can generate the required code and we can include it in our app.

For iOS, we need to create a `podspec` file which will define the Component as a dependency.
The `podspec` file for our Component will look like this

```ruby title="rnt-centered-text.podspec"
require "json"

package = JSON.parse(File.read(File.join(__dir__, "package.json")))

folly_version = '2021.06.28.00-v2'
folly_compiler_flags = '-DFOLLY_NO_CONFIG -DFOLLY_MOBILE=1 -DFOLLY_USE_LIBCPP=1 -Wno-comma -Wno-shorten-64-to-32'

Pod::Spec.new do |s|
  s.name            = "rnt-centered-text"
  s.version         = package["version"]
  s.summary         = package["description"]
  s.description     = package["description"]
  s.homepage        = package["homepage"]
  s.license         = package["license"]
  s.platforms       = { :ios => "11.0" }
  s.author          = package["author"]
  s.source          = { :git => package["repository"], :tag => "#{s.version}" }

  s.source_files    = "ios/**/*.{h,m,mm,swift}"

  s.dependency "React-Core"

  s.compiler_flags = folly_compiler_flags + " -DRCT_NEW_ARCH_ENABLED=1"
  s.pod_target_xcconfig    = {
    "HEADER_SEARCH_PATHS" => "\"$(PODS_ROOT)/boost\"",
    "OTHER_CPLUSPLUSFLAGS" => "-DFOLLY_NO_CONFIG -DFOLLY_MOBILE=1 -DFOLLY_USE_LIBCPP=1",
    "CLANG_CXX_LANGUAGE_STANDARD" => "c++17"
  }

  s.dependency "React-RCTFabric"
  s.dependency "React-Codegen"
  s.dependency "RCT-Folly", folly_version
  s.dependency "RCTRequired"
  s.dependency "RCTTypeSafety"
  s.dependency "ReactCommon/turbomodule/core"
end
```

The `podspec` file has to be a sibling of the `package.json` file and its name is the one we set in the `package.json`'s `name` property: `rnt-centered-text`.

The first part of the file prepares some variables we will use throughout the rest of it:

- the `package` variable which contains information read from the `package.json`
- the `folly_version` variable to set the version of `folly` we need.
- the `folly compiler_flags` that we need to set in the new architecture.

The next section contains some information used to configure the pod, like its name, version, a description and so on.

Finally, we have a set of dependencies that are required by the new architecture.

#### Android: Create the `build.gradle` file

For what concerns Android, we need to create a `build.gradle` file in the `android` folder. The file will have the following shape

```kotlin title="build.gradle"
buildscript {
  ext.safeExtGet = {prop, fallback ->
    rootProject.ext.has(prop) ? rootProject.ext.get(prop) : fallback
  }
  repositories {
    google()
    gradlePluginPortal()
  }
  dependencies {
    classpath("com.android.tools.build:gradle:7.2.0")
  }
}

apply plugin: 'com.android.library'
apply plugin: 'com.facebook.react'

android {
  compileSdkVersion safeExtGet('compileSdkVersion', 31)

  defaultConfig {
    minSdkVersion safeExtGet('minSdkVersion', 21)
    targetSdkVersion safeExtGet('targetSdkVersion', 31)
    buildConfigField "boolean", "IS_NEW_ARCHITECTURE_ENABLED", true
  }
}

repositories {
  maven {
    // All of React Native (JS, Obj-C sources, Android binaries) is installed from npm
    url "$projectDir/../node_modules/react-native/android"
  }
  mavenCentral()
  google()
}

dependencies {
  implementation 'com.facebook.react:react-native:+'
}
```

_TODO: Briefly explain the various `build.gradle` section if this add some value to the guide_

### Native Code

#### iOS

#### Android

### Adding the Fabric Component To Your App
