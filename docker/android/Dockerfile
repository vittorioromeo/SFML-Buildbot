FROM debian:stable-slim AS final-layer

RUN DEBIAN_FRONTEND=noninteractive \
    ln -fs /usr/share/zoneinfo/Europe/Berlin /etc/localtime \
    && apt-get update && apt-get install --no-install-recommends -y \
    tzdata \
    wget \
    unzip \
    git \
    build-essential \
    python3-pip \
    python3-dev \
    default-jre \
    && dpkg-reconfigure --frontend noninteractive tzdata \
    && apt-get autoremove -y \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# CMake
RUN wget https://github.com/Kitware/CMake/releases/download/v3.22.1/cmake-3.22.1-linux-x86_64.sh \
    && sh cmake-3.22.1-linux-x86_64.sh --skip-license --prefix=/usr/local \
    && rm cmake-3.22.1-linux-x86_64.sh

# Buildbot Worker
RUN pip3 install 'buildbot-worker' \
    && rm -rf ~/.cache

# Gradle
RUN wget https://services.gradle.org/distributions/gradle-7.3.1-bin.zip \
    && unzip gradle-7.3.1-bin.zip \
    && mv gradle-7.3.1 /opt \
    && rm -rf gradle-*.zip

ENV PATH "/opt/gradle-7.3.1/bin:$PATH"

# Android SDK/NDK
ENV ANDROID_HOME "/opt/android-sdk"

RUN wget https://dl.google.com/android/repository/commandlinetools-linux-7583922_latest.zip \
    && unzip commandlinetools-linux-7583922_latest.zip \
    && echo "y" | cmdline-tools/bin/sdkmanager --sdk_root=$ANDROID_HOME --install "cmdline-tools;latest" \
    && rm -rf cmdline-tools \
    && rm -rf commandlinetools-linux* \
    && rm -rf ~/.android/cache

ENV PATH "$ANDROID_HOME/cmdline-tools/latest/bin:$PATH"

RUN echo "y" | sdkmanager --install "ndk;21.4.7075529" \
    && rm -rf ~/.android/cache

ENV ANDROID_NDK_HOME "/opt/android-sdk/ndk/21.4.7075529"

# Test Build
FROM final-layer as test-layer

RUN wget https://github.com/SFML/SFML/archive/refs/heads/master.zip \
    && unzip master.zip \
    && cd SFML-master \
    && mkdir build \
    && sed -i s@\${CMAKE_ANDROID_NDK}/sources/third_party@$PWD/build/install@g CMakeLists.txt \
    && sed -i s@\${CMAKE_ANDROID_NDK}/sources/third_party@$PWD/build/install@g cmake/Config.cmake \
    && sed -i s@third_party/sfml@sfml@g examples/android/app/src/main/jni/Android.mk \
    && sed -i s@third_party/sfml@sfml@g src/SFML/Android.mk \
    && sed -i s@gradle:3.0.0@gradle:7.0.0@g examples/android/build.gradle \
    && sed -i s@targetSdkVersion\ 19@targetSdkVersion\ 29@g examples/android/app/build.gradle \
    && cd build \
    && cmake -DCMAKE_ANDROID_ARCH_ABI=armeabi-v7a -DCMAKE_SYSTEM_NAME=Android -DSFML_BUILD_TEST_SUITE=FALSE -DCMAKE_ANDROID_NDK=$ANDROID_NDK_HOME -DCMAKE_ANDROID_NDK_TOOLCHAIN_VERSION=clang -DCMAKE_ANDROID_STL_TYPE=c++_shared -DCMAKE_ANDROID_API=26 -DCMAKE_BUILD_TYPE=Debug -DBUILD_SHARED_LIBS=TRUE .. && cmake --build . -j8 -- install \
    && cmake -DCMAKE_ANDROID_ARCH_ABI=armeabi-v7a -DCMAKE_SYSTEM_NAME=Android -DSFML_BUILD_TEST_SUITE=FALSE -DCMAKE_ANDROID_NDK=$ANDROID_NDK_HOME -DCMAKE_ANDROID_NDK_TOOLCHAIN_VERSION=clang -DCMAKE_ANDROID_STL_TYPE=c++_shared -DCMAKE_ANDROID_API=26 -DCMAKE_BUILD_TYPE=Debug -DBUILD_SHARED_LIBS=FALSE .. && cmake --build . -j8 -- install \
    && cmake -DCMAKE_ANDROID_ARCH_ABI=armeabi-v7a -DCMAKE_SYSTEM_NAME=Android -DSFML_BUILD_TEST_SUITE=FALSE -DCMAKE_ANDROID_NDK=$ANDROID_NDK_HOME -DCMAKE_ANDROID_NDK_TOOLCHAIN_VERSION=clang -DCMAKE_ANDROID_STL_TYPE=c++_shared -DCMAKE_ANDROID_API=26 -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=TRUE .. && cmake --build . -j8 -- install \
    && cmake -DCMAKE_ANDROID_ARCH_ABI=armeabi-v7a -DCMAKE_SYSTEM_NAME=Android -DSFML_BUILD_TEST_SUITE=FALSE -DCMAKE_ANDROID_NDK=$ANDROID_NDK_HOME -DCMAKE_ANDROID_NDK_TOOLCHAIN_VERSION=clang -DCMAKE_ANDROID_STL_TYPE=c++_shared -DCMAKE_ANDROID_API=26 -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=FALSE .. && cmake --build . -j8 -- install \
    && cd ../examples/android \
    && NDK_MODULE_PATH=$PWD/../../build/install gradle --stacktrace assembleDebug \
    && NDK_MODULE_PATH=$PWD/../../build/install gradle --stacktrace assembleRelease \
    && cp app/build/outputs/apk/debug/*-debug.apk /debug-verification.apk \
    && cp app/build/outputs/apk/release/*-release-unsigned.apk /release-verification.apk

# Finalize Image
FROM final-layer

COPY --from=test-layer /debug-verification.apk /debug-verification.apk
COPY --from=test-layer /release-verification.apk /release-verification.apk

RUN echo ' \n\
rm -rf $BUILDBOT_HOME \n\
mkdir -p $BUILDBOT_HOME \n\
 \n\
buildbot-worker create-worker $BUILDBOT_HOME $BUILDBOT_MASTER_HOST:$BUILDBOT_MASTER_PORT $BUILDBOT_WORKER_NAME $BUILDBOT_WORKER_PASSWORD \n\
echo $BUILDBOT_ADMIN > $BUILDBOT_HOME/info/admin \n\
echo $BUILDBOT_DESCRIPTION > $BUILDBOT_HOME/info/host \n\
 \n\
buildbot-worker start --nodaemon $BUILDBOT_HOME \n\
' > /start.sh \
    && chmod +x /start.sh

ENV BUILDBOT_HOME=/home/buildbot-worker
ENV BUILDBOT_ADMIN="Your Name Here <admin@youraddress.invalid>"
ENV BUILDBOT_DESCRIPTION="Please put a description of this build host here"
ENV BUILDBOT_MASTER_HOST="masteraddress.invalid"
ENV BUILDBOT_MASTER_PORT=9989
ENV BUILDBOT_WORKER_NAME="buildbot-worker-name"
ENV BUILDBOT_WORKER_PASSWORD="buildbot-worker-password"

CMD /start.sh
