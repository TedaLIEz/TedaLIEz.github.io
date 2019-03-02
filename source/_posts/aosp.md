---
title: AOSP MacOS编译问题总结
---

1. AS反复提示indexing

   https://github.com/flutter/flutter-intellij/issues/1735#issuecomment-376597481 在flutter中找到一个办法，在AS 3.1中对应的iml配置项下加一行

   ```xml
   <option name="ALLOW_USER_CONFIGURATION" value="false" />
   ```

2. "sed: illegal option -- r"

   mac上安装`brew install gnu-sed --with-default-names`之后`export PATH="/usr/local/opt/gnu-sed/libexec/gnubin:$PATH"`

3. repo init -u <https://github.com/LineageOS/android.git> -b lineage-15.2

4. 获取 proprietary blobs的相关文件，在.repo/local_manifests/roomservice.xml(如果没有手动创建一个), 

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <manifest>
     <project name="LineageOS/android_device_oneplus_oneplus3" path="device/oneplus/oneplus3" remote="github" />
     <project name="LineageOS/android_kernel_oneplus_msm8996" path="kernel/oneplus/msm8996" remote="github" />
     <project name="LineageOS/android_packages_resources_devicesettings" path="packages/resources/devicesettings" remote="github" />
     <project depth="1" name="TheMuppets/proprietary_vendor_oneplus" path="vendor/oneplus" />
     <project name="LineageOS/android_device_oppo_common" path="device/oppo/common" remote="github" />
   </manifest>
   ```

5. repo sync
6. 未完待续，后续补充

