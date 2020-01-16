---
title: 使用SimpleITK读取DICOM
date: 2019-06-08 19:38:01
tags: [SimpleITK,DICOM,Python]
categories: DICOM
---

SimpleITK是ITK的简化接口，专门用于处理医学影像，在SimpleITK中，图像的概念与我们在计算机视觉中常用的RGB图像差异很大，后者只是一个多维矩阵，是一个数学上的概念，而在SimpleITK中，图像是一种物理实体，图像中的每一个像素都是物理空间中的一个点，不光有着像素值，还有着坐标，间距，方向等概念。本文将介绍python版SimpleITK读取DICOM并从DICOM study中提取需要的序列的方法。

<!--more-->

## 安装
`pip install SimpleITK`

## 读取DICOM影像

- 读取单张DICOM影像

```python
import SimpleITK as sitk

file = sitk.ReadImage(filepath)
 
 # 图像大小
file.GetSize()
# 坐标原点
file.GetOrigin()
# 像素间距
file.GetSpacing()
# 方向
file.GetDirection()

# 获取影像元数据(返回DICOM tags元组)
file.GetMetaDataKeys()

# 像素矩阵
pixel_array = sitk.GetArrayFromImage(file)
```

- 读取DICOM序列

```python
import os
import SimpleITK as sitk

path = "/dicom_study_dir"

reader = sitk.ImageSeriesReader()
seriesIDs = reader.GetGDCMSeriesIDs(path)
dicom_names = reader.GetGDCMSeriesFileNames(path, seriesID[0])
reader.SetFileNames(dicom_names)
image = reader.Execute()

```

## 读取特定序列并保存为NIfTI格式
医学影像处理时，常常需要三维格式的数据，可以利用SimpleITK从DICOM study文件夹中提取需要的序列，并写入NIfTI文件。以下为示例：
```python
import logging
import os
import SimpleITK as sitk

# 从DICOM影像中读取Tag信息
def get_tag(dcm, key, default_value=''):
    try:
        value = dcm.GetMetaData(key).strip()
    except Exception as e:
        logging.error(e)
        value = default_value

    return value

# 从DICOM study文件夹中提取需要的序列并转换
def extract_series(study_dir, series_desc):
    # 获取所有series id,返回一个元组
    series_ids = sitk.ImageSeriesReader.GetGDCMSeriesIDs(study_dir)
    for series_id in series_ids:
        # 获取该序列的所有DICOM影像，返回一个包含该序列所有影像路径的元组
        files = sitk.ImageSeriesReader.GetGDCMSeriesFileNames(study_dir, series_id, recursive=False, useSeriesDetails=False)
        reader = sitk.ImageSeriesReader()
        reader.SetFileNames(files)
        try:
            img = reader.Execute()
        except RuntimeError as e:
            # 有的DICOM影像将一个序列所有影像都放在同一张DICOM文件中（MultiFrame），读取时会出现下面这个异常
            if 'The file in the series have unsupported 3 dimensions' in str(e):
                img = sitk.ReadImage(files[0])
            else:
                logging.error('Error in reading image, dir %s, series id %s, detail message: %s' % (study_dir, series_id, e))
                continue
		# 从当前序列取一张影像，便于读取其Tag信息                
        dcm = sitk.ReadImage(files[0])
        # 获取当前序列的序列描述
        series_description = get_tag(dcm, '0008|103e')
        # 判断是否是需要的序列
        if series_description == series_desc:
            nii_name = str(series_id) + "nii.gz"
            # 写入nii.gz文件
            sitk.WriteImage(img, os.path.join(study_dir, nii_name))
        else:
            continue

```

