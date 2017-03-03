---
published: true
layout: post
title: Install-OpenCV3.2.0-with-Anaconda3-Python3.5.2-on-MacOS
category: Python
tags: Python OpenCV3 Anaconda3 macOS
time: 2017.03.03 14:22:00
keywords: 
description: 在macOS上为anaconda3(python3.5.2)安装OpenCV3

---

#### macOS system information

1. macOS Sierra 10.12.3 
2. python 版本: 3.5.2
3. 要安装的opencv3版本: 3.2.0

  
#### 说明

  安装的时候也是各种问题, 折腾了三四天呢, 主要是 各种依赖 和 library 路径问题, 也在`stackoverflow` 上查了很多资料, 也试过 `brew`和`conda`的安装方式, 都各种错误,依赖找不到,
最后还是用源码编译的方式安装的, 下面贴出我的系统`cmake`命令以及一些参考文档,希望有用. (建议先看下 [http://www.cnblogs.com/beer/p/5668620.html](http://www.cnblogs.com/beer/p/5668620.html)此文章,注意里面的每个注意项,
然后再结合自己的环境编译,在`stackoverflow上`查找解决问题)
  
#### cmake 命令

      cmake -DCMAKE_BUILD_TYPE=RELEASE \
      -DCMAKE_INSTALL_PREFIX=/usr/local \
      -DOPENCV_EXTRA_MODULES_PATH=/Users/kute/work/gitwork/opencv_contrib/modules \
      -DPYTHON3_LIBRARY=/Users/kute/anaconda3/lib/libpython3.5m.dylib \
      -DPYTHON3_INCLUDE_DIR=/Users/kute/anaconda3/include/python3.5m \
      -DPYTHON3_EXECUTABLE=/Users/kute/anaconda3/bin/python3 \
      -DBUILD_opencv_python2=OFF -DBUILD_opencv_python3=ON \
      -DINSTALL_PYTHON_EXAMPLES=ON -DINSTALL_C_EXAMPLES=OFF \
      -DBUILD_EXAMPLES=ON \
      -DPYTHON3_NUMPY_INCLUDE_DIRS=/Users/kute/anaconda3/lib/python3.5/site-packages/numpy/core/include \
      -D PYTHON3_PACKAGES_PATH=/Users/kute/anaconda3/lib/python3.5/site-packages \
      -DCMAKE_OSX_ARCHITECTURES=x86_64 \
      -DENABLE_PRECOMPILED_HEADERS=OFF \
      -DBUILD_opencv_text=OFF \
      -DBUILD_opencv_tracking=OFF \
      -DBUILD_opencv_legacy=OFF ..
    
#### 参考链接

  1. [http://www.cnblogs.com/beer/p/5668620.html](http://www.cnblogs.com/beer/p/5668620.html)

  2. [http://stackoverflow.com/questions/23119413/how-to-install-python-opencv-through-conda](http://stackoverflow.com/questions/23119413/how-to-install-python-opencv-through-conda)
  3. [http://stackoverflow.com/questions/37110600/how-to-install-opencv3-in-anaconda3-offline](http://stackoverflow.com/questions/37110600/how-to-install-opencv3-in-anaconda3-offline)
  4. [http://stackoverflow.com/questions/41873941/cant-install-opencv3-on-anaconda3-python3-6-on-macos](http://stackoverflow.com/questions/41873941/cant-install-opencv3-on-anaconda3-python3-6-on-macos)
  5. [http://tsaith.github.io/install-opencv-3-for-python-3-on-osx.html](http://tsaith.github.io/install-opencv-3-for-python-3-on-osx.html)
  6. [http://www.pyimagesearch.com/2015/06/29/install-opencv-3-0-and-python-3-4-on-osx/](http://www.pyimagesearch.com/2015/06/29/install-opencv-3-0-and-python-3-4-on-osx/)
  7. [http://stackoverflow.com/questions/41873941/cant-install-opencv3-on-anaconda3-python3-6-on-macos](http://stackoverflow.com/questions/41873941/cant-install-opencv3-on-anaconda3-python3-6-on-macos)
  8. [http://stackoverflow.com/questions/21804384/install-opencv-using-macports-on-mac-os-x-10-9-mavericks](http://stackoverflow.com/questions/21804384/install-opencv-using-macports-on-mac-os-x-10-9-mavericks)
  9. [https://github.com/opencv/opencv/issues/8303](https://github.com/opencv/opencv/issues/8303)
  10. [http://stackoverflow.com/questions/39575800/how-do-i-get-opencv-for-python3-with-anaconda3](http://stackoverflow.com/questions/39575800/how-do-i-get-opencv-for-python3-with-anaconda3)
  
  11. [https://github.com/opencv/opencv/issues](https://github.com/opencv/opencv/issues)
  12. [https://github.com/opencv/opencv_contrib/issues](https://github.com/opencv/opencv_contrib/issues)
  
  
  