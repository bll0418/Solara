name: Android CI/CD

on:
  push:
    branches: [ "main", "master" ]
  workflow_dispatch:

jobs:
  build:
    # 使用 macOS 环境通常能解决 Linux 上的文件锁冲突
    runs-on: macos-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      # 1. 适配项目结构：将纯前端文件移动到 www 目录
      - name: Prepare Web Assets
        run: |
          mkdir -p www
          mv css www/ || true
          mv js www/ || true
          mv *.html www/ || true
          # 【关键】将 functions 目录放到 android 打包范围外，因为 WebView 无法运行它
          # 此处保持 functions 在根目录即可，Capacitor 不会把它放入 www
          if [ ! -f www/index.html ]; then echo "index.html not found in www"; exit 1; fi

      # 2. 安装 Capacitor 并初始化
      - name: Install and Init Capacitor
        run: |
          npm install @capacitor/core @capacitor/cli @capacitor/android
          rm -rf android
          npx cap init Solara com.example.solara --web-dir www
          npx cap add android

      # 3. 构建 APK (激进修复区)
      - name: Build APK
        run: |
          cd android
          # 【关键】强制禁用守护进程
          echo "org.gradle.daemon=false" > gradle.properties
          
          # 【关键】清理损坏的缓存文件
          rm -rf ~/.gradle/caches/
          rm -rf .gradle/
          
          # 【关键】执行编译，禁用配置缓存以解决 Jar 文件创建失败问题
          ./gradlew assembleDebug --no-daemon --no-parallel -Dorg.gradle.unsafe.configuration-cache=false
                
      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: release-apk
          path: android/app/build/outputs/apk/debug/app-debug.apk
