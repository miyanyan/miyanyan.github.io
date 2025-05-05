---
title: 如何给vcpkg提pr
description: 在github上给vcpkg提pr问题记录
date: 2024-11-17
categories:
    - vcpkg
tags:
    - vcpkg
    - github
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

# 如何给vcpkg提pr

笔者偶尔会给vcpkg提交一些包的更新，这里记录下遇到的问题，与大家分享，如能起到一点点帮助那我就更开心了。

首先是github提pr的基本流程，这里假设大家都已经知道了，此为前提。

## 更新包

### 更新版本

以[fmt](https://github.com/microsoft/vcpkg/pull/39738)为例，我们这次的目的要更新fmt的版本号到`11.0.2`

1. 在vcpkg的路径ports文件夹下找到要更新的包，为例，那么我们就要找到文件夹`vcpkg/ports/fmt`
2. 在这个文件夹下有两个主要文件，也就是我们要修改的地方
   * `vcpkg.json` 这个文件对fmt包进行了基本的描述，因为我们主要关注版本的更新，所以我们主要看参数`version`和`port-version`  
      `version`对应版本号，把它更新为`11.0.2`; `port-version`为当前版本的修改号，一般的，当我们更新版本时要去掉`port-version`(port-version将在下一节修复问题时用到)

      好了，更新完后应该是这样子的

      ``` json
      {
        "name": "fmt",
        "version": "11.0.2", # <--- 更新了这里的版本号。VVV 下面没有port-version
        "description": "{fmt} is an open-source formatting library providing a fast and safe alternative to C stdio and C++ iostreams.",
        "homepage": "https://github.com/fmtlib/fmt",
        "license": "MIT",
        "dependencies": [
          {
            "name": "vcpkg-cmake",
            "host": true
          },
          {
            "name": "vcpkg-cmake-config",
            "host": true
          }
        ]
      }
      ```

    * `portfile.cmake` 这个文件对fmt怎么用cmake构建进行了描述，因为我们主要关注版本的更新，所以我们只看文件里的`vcpkg_from_github`函数，这个函数里有两个要改的变量
      * `REF` vcpkg是根据`REF`的值去判断下载github上的哪个版本的release去编译的，如果你发现`REF`的值是`"${VERSION}"`，
      那么这里一般就不需要改了，否则你需要把它改成版本号(当然，你提pr的时候就会发现vcpkg的官方人员会提醒你使用`"${VERSION}"`，减少一处改动何乐而不为呢哈哈)
      * `SHA512` 这个值是github上release的压缩包的哈希值，用于校验下载的文件是否正确。那么怎么得到这个哈希值呢?
        * 在vcpkg的根目录下执行`vcpkg install fmt`， 等待一段时间，vcpkg会报错，在报错信息里找到这两行：

          ``` cmake
          ...
          Expected hash: xxxxx
          Actual hash: 47ff6d289dcc22681eea6da465b0348172921e7cafff8fd57a1540d3232cc6b53250a4625c954ee0944c87963b17680ecbc3ea123e43c2c822efe0dc6fa6cef3
          ...
          ```

          看到没，这个Actual hash的值就是你要修改的`SHA512`的值了，复制过来。
        * 在vcpkg的根目录下再次执行`vcpkg install fmt`，如果没有报错那么恭喜你，更新包的流程就完成了一大半了！

      好了，更新完后应该是这样子的

      ``` cmake
        vcpkg_from_github(
          OUT_SOURCE_PATH SOURCE_PATH
          REPO fmtlib/fmt
          REF "${VERSION}" # <--- 无需修改
          SHA512 47ff6d289dcc22681eea6da465b0348172921e7cafff8fd57a1540d3232cc6b53250a4625c954ee0944c87963b17680ecbc3ea123e43c2c822efe0dc6fa6cef3 # <--- 必须修改哈希值
          HEAD_REF master
      )
      ```

3. 将刚才的修改git commit
4. 在vcpkg的根目录下执行`vcpkg x-add-version --all`，这会修改vcpkg用于记录的数据库，对应两个文件
   * `versions/f-/fmt.json`
   * `versions/baseline.json`
   当然，这两个文件的改动是自动修改的，再次git commit
5. 将git修改提交到vcpkg的pull request中

### 修复当前版本的问题

很多时候不是简简单单更新个版本号就能正确更新的，某些包的更新可能会导致依赖于此包的其他包编译报错，某些包的更新可能会引入自己的新bug等等...

我们还是以[fmt](https://github.com/microsoft/vcpkg/pull/39738)为例，当fmt从v10升级到v11后，有很多大的改动，
比如要使用fmt的join函数，必须要显式的`#include <fmt/ranges.h>`，这会导致libtorch编译报错，我们需要修改libtorch相关的报错文件，在其文件中`#include <fmt/ranges.h>`

那么要怎么做呢，我们的核心思想就是对这次修复形成一个commit，利用git的patch机制生成一个patch文件，最终在构建时将这个patch文件引入进来，从而解决编译报错等问题。

参考[https://github.com/Microsoft/vcpkg-docs/blob/main/vcpkg/examples/patching.md](https://github.com/Microsoft/vcpkg-docs/blob/main/vcpkg/examples/patching.md)

1. 为libtorch创建git本地仓库
   * 执行`vcpkg install libtorch`，此时由于fmt的更新会导致编译报错，没错就是缺少`#include <fmt/ranges.h>`导致的报错。我们找到vcpkg下载下来的libtorch的源文件目录  
     `vcpkg/buildtrees/libtorch/src`这个目录下存储着libtorch每次的下载缓存，一般的我们选最新的那个目录并进入该目录
   * 执行`git init .`初始化本地仓库
   * 执行`git add .`把libtorch的所有原文件加入暂存区
   * 执行`git commit -m "temp"`把所有原文件提交
  
2. 修改libtorch的编译报错问题
  这里我们已经知道了要显式的`#include <fmt/ranges.h>`，直接找到对应的文件修改吧！

3. 为修改完的文件生成patch
   * 执行`git status`可以看到我们修改的文件，确认修改无误
   * 执行`git diff --ignore-space-at-eol | out-file -enc ascii ..\..\..\..\ports\libtorch\fix-build-error-with-fmt11.patch`即可把patch文件生成到`vcpkg/ports/libtorch`下，
     这个文件长这样[fix-build-error-with-fmt11.patch](https://github.com/microsoft/vcpkg/pull/39738/files#diff-20152fea9680d25580927f0beb5e3353810aef44c5dfe4c505e5c2a46f60b699)  
     注意，vcpkg的官方人员会要求你文件用`LF`而不是`CRLF`，记得确定

4. 在`portfile.cmake`文件中添加该patch

   ``` cmake
    vcpkg_from_github(
        # 其他参数...
        PATCHES
            fix-build-error-with-fmt11.patch # <-- 这里就是生成的patch文件了
    )
    ```

5. 验证行patch是否能解决问题
  再次执行`vcpkg install libtorch`，如果没有报错那么就是成功了！

6. 将`vcpkg.json`文件中的`port-version`号加1，如果没有`port-version`则说明当前的`port-version`号为0

7. 将刚才的修改git commit

8. 在vcpkg的根目录下执行`vcpkg x-add-version --all`，再次git commit

9. 将git修改提交到vcpkg的pull request中

## 添加包

如果你已经知道了如何更新包，那么你肯定熟悉了vcpkg的目录结构，这里我们再复述一下，vcpkg目录下的`ports/<your-port-name>`主要包含两个文件：

* portfile.cmake: 如何用cmake编译
* vcpkg.json: 这个包的依赖关系等描述

接下来我们以添加[ucoro](https://github.com/microsoft/vcpkg/pull/42969)到vcpkg为例

1. 在ports文件夹下新建ucoro文件夹
2. 在ucoro文件夹下添加portfile.cmake和vcpkg.json两个文件。(我一般是找个就近的包把它的这俩文件copy过来)
3. 修改vcpkg.json文件内容，主要是名称、版本号、描述、网址、许可证、依赖
4. 修改portfile.cmake文件，一般的，我们把下面的每一个函数都调用一下总不会错
   * 如果是header-only的库，那么就在第一行添加`set(VCPKG_BUILD_TYPE release)`，这样不会有debug的文件目录生成，ucoro就是一个header-only的库，这里我加上了这一行。
   * 使用[vcpkg_from_github](https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_from_github)从github下载ucoro的release文件
   * 使用[vcpkg_cmake_configure](https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_cmake_configure)传入项目编译的cmake参数
   * 使用[vcpkg_cmake_install](https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_cmake_install)调用cmake install命令安装库
   * 使用[vcpkg_cmake_config_fixup](https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_cmake_config_fixup)确保生成的cmake安装文件符合vcpkg的要求
   * 使用[vcpkg_fixup_pkgconfig](https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_fixup_pkgconfig)修复 *.pc 文件中的通用路径
   * 使用`file(INSTALL "${SOURCE_PATH}/LICENSE_1_0.txt" DESTINATION "${CURRENT_PACKAGES_DIR}/share/${PORT}" RENAME copyright)`复制ucoro的license到目标目录
5. 好了，现在就和上面的更新包操作一样，使用`vcpkg install ucoro`直到vcpkg不再报错和警告即大功告成，最终的port.cmake文件如下，可以看到我还添加了一个cmake-install.patch文件，这是因为ucoro的cmake文件里没有写cmake install指令，当然cmake语法这里不做介绍。

   ``` cmake
    set(VCPKG_BUILD_TYPE release)  # header-only

    vcpkg_from_github(
        OUT_SOURCE_PATH SOURCE_PATH
        REPO avplayer/ucoro
        REF "v${VERSION}"
        SHA512 c3436b436ef1ebb3d47a65db9603842293bdb6451bc6fb738a63d61a63b52901e223f46625d956303566dc52dfb38ffb2c6ce20016c18b444f9cb3e2e701e613
        HEAD_REF main
        PATCHES
            cmake-install.patch
    )

    vcpkg_cmake_configure(
        SOURCE_PATH "${SOURCE_PATH}"
        OPTIONS
            -DUCORO_BUILD_TESTING=OFF
    )

    vcpkg_cmake_install()
    vcpkg_cmake_config_fixup()
    vcpkg_fixup_pkgconfig()

    file(INSTALL "${SOURCE_PATH}/LICENSE_1_0.txt" DESTINATION "${CURRENT_PACKAGES_DIR}/share/${PORT}" RENAME copyright)
    ```

6. 和更新包的操作一样，git commit并提交pr吧！  

## 常见问题

### 1. 有的包生成的.pc文件里有绝对路径，导致vcpkg的ci报错

   使用[vcpkg_fixup_pkgconfig()](https://learn.microsoft.com/zh-cn/vcpkg/maintainers/functions/vcpkg_fixup_pkgconfig)进行修复，如[更新magic-enum](https://github.com/microsoft/vcpkg/pull/42158)时我发现之前的portfile.cmake里并未对pc文件进行修复

### 2. 某个包的当前版本的bug需要修复，如何添加patch

* 这个bug已经被修复了，只是没有更新到这个版本

  * 首先我们找到这个bug修复对应的commit，这里我们以[更新fmt到11](https://github.com/microsoft/vcpkg/pull/39738)为例，在这个pr里我发现fmt的更新导致folly编译报错，具体原因忽略，这个编译报错被修复对应的commit为[https://github.com/facebook/folly/commit/21e8dcd464ee46b2144a1e4d4c0e452355ae15f0](https://github.com/facebook/folly/commit/21e8dcd464ee46b2144a1e4d4c0e452355ae15f0)，我们进入到这个commit对应的链接，可以看到修改的git diff。

  * 但是！咱们现在要的是一个可以用的patch文件，这里就需要加入一点github官方文档未提供的方法了(也许提供了但是我没找到？)，在这个commit链接后面加上`.patch`，即[https://github.com/facebook/folly/commit/21e8dcd464ee46b2144a1e4d4c0e452355ae15f0.patch](https://github.com/facebook/folly/commit/21e8dcd464ee46b2144a1e4d4c0e452355ae15f0.patch)，可以发现现在是纯文本了，接下来还要在最后加上`?full_index=1`，按讨论区的说法，加上这个之后可以保证链接的稳定。
    > You have to add ?full_index=1 to the URL to get a full commit hash in the patch, otherwise the number of characters used to write the commit in the patch will grow as the project grows.

    参考[https://github.com/orgs/community/discussions/46034#discussioncomment-4846112](https://github.com/orgs/community/discussions/46034#discussioncomment-4846112)
  * patch文件搞定了，接下来就是把这个链接放到vcpkg的portfile.cmake文件中，调用[vcpkg_download_distfile](https://github.com/microsoft/vcpkg/pull/39738/files#diff-43a3685f2f444b4804fbb55de14055069de61e824ca3fc9ae9b27fb5ffd16a32R8-R13)把文件下载下来，

    ``` cmake
    vcpkg_download_distfile(FMT11_RANGE_PATCH # 这里给patch起个名称
        URLS https://github.com/facebook/folly/commit/21e8dcd464ee46b2144a1e4d4c0e452355ae15f0.patch?full_index=1
        FILENAME fmt11-range.patch
        SHA512 6a3afe361cd24b4f62b3aba625dfbbfb767c91f27fa45ed4604adc5ec3d574e571ece13eeda0d9d47b8a37166fc31b1ed7f58f120a35d35977085a08172de105
    )
    ```

    并在[vcpkg_from_github](https://github.com/microsoft/vcpkg/pull/39738/files#diff-43a3685f2f444b4804fbb55de14055069de61e824ca3fc9ae9b27fb5ffd16a32R27)中设置这个文件为对应的patch。至此完成了拉取上游修改并添加为patch的操作。

    ``` cmake
    vcpkg_from_github(
        OUT_SOURCE_PATH SOURCE_PATH
        REPO facebook/folly
        REF "v${VERSION}"
        SHA512 f129d4e530b5c8aaf4cbfb0c813e84ee911ec26e43fa01e6e1c9557501c605a8123d46d0c689b32eb5e5a57280968662a5fa370ede17af7526db59545e9a70db
        HEAD_REF main
        PATCHES
            disable-non-underscore-posix-names.patch
            fix-windows-minmax.patch
            fix-deps.patch
            disable-uninitialized-resize-on-new-stl.patch
            fix-unistd-include.patch
            fix-fmt11-cmake.patch
            ${FMT11_RANGE_PATCH} # 这里就是下载下来的patch
    )
    ```

### 3. 如何在vcpkg的ci检查中跳过某些检查

在[更新fastdds](https://github.com/microsoft/vcpkg/pull/40984)时我遇到了fastdds在老版本的android上由于缺少依赖编译不过去的问题

官方的回复也是说针对这种情况要在ci里跳过检查
> CI is instructed to skip many ports which required API level 24 for android

主要修改的地方为文件`scripts/ci.baseline.txt`
在这个文件里添加希望跳过的检查：

``` plain text
fastdds:arm-neon-android=fail
fastdds:arm64-android=fail
fastdds:x64-android=fail
```
