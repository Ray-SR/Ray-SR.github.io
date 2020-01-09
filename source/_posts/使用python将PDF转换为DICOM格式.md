---
title: 使用python将PDF转换为DICOM格式
date: 2019-06-25 10:38:01
tags: [DICOM,Python]
categories: DICOM
---

由于公司业务需要，尝试将AI辅助诊断报告（PDF格式）转换为DICOM格式，这样，当DICOM影像上传到云平台计算完成后，可以将生成的AI辅助诊断报告推送到医院PACS系统，医生就可以在DICOM viewer中同时看到病人原始影像和AI报告。

<!--more-->

## 思路
DICOM影像通常分为三个层级：`Study`、`Series`、`Instance`，其中Study代表病人一次检查的全部影像，通常放在一个文件夹内，而Series则代表这次检查中的不同影像序列，每个序列的扫描参数、用途各有差异，而Instance则是代表每张影像的实体。为了区分影像，DICOM标准中对于Study、Series、Intance的序号（UID）命名有着明确的规范，另外对于影像的编码格式、传输语法以及影像信息都有要求，在创建DICOM影像时都要注意。

DICOM tag信息主要分为三大部分，第一部分File Meta文件头，指定了影像最重要的基本信息，包含编码格式、传输语法等信息，第二部分则是常规的病人信息、影像信息字段，而影像的像素数据则保存在第三部分`Pixel Data`字段中。

为了将PDF报告转换成DICOM格式，可以先利用`pdf2image`将PDF转换为常见的图片格式，再将图片转换为`bytes`类型数据，写入DICOM tag中的`Pixel Data`字段，同时还需要给这些创建的DICOM影像指定UID、编码格式、传输语法等Tag信息。

## 代码
以下是PDF转DICOM的参考代码，注意这里的应用情况是将PDF转换后的DICOM影像写入到原始DICOM文件夹中，支持含有多页的PDF文件

```python
import os
import tempfile
import datetime
import pydicom
from pydicom.dataset import Dataset, FileDataset
from pdf2image import convert_from_path


# 从DICOM study文件夹中读取影像，返回最后一张（InstanceUID最大的一张）影像信息的字典
def read_dcm(dicom_dir):
    dicom_list = os.listdir(dicom_dir)
    info_list = []
    for dicom in dicom_list:
        # 使用pydicom读取DICOM影像信息
        ds = pydicom.dcmread(os.path.join(dicom_dir, dicom))

        info_dict = {
            'MediaStorageSOPInstanceUID': ds.file_meta.MediaStorageSOPInstanceUID,
            'MediaStorageSOPClassUID': ds.file_meta.MediaStorageSOPClassUID,
            'ImplementationClassUID': ds.file_meta.ImplementationClassUID,
            'PatientName': ds.PatientName,
            'PatientID': ds.PatientID,
            'PatientSex': ds.PatientSex,
            'PatientBirthDate': ds.PatientBirthDate,
            'PatientAge': ds.PatientAge,
            'ContentDate': ds.ContentDate,
            'ContentTime': ds.ContentTime,
            'SOPClassUID': ds.SOPClassUID,
            'SOPInstanceUID': ds.SOPInstanceUID,
            'StudyInstanceUID': ds.StudyInstanceUID,
            'SeriesInstanceUID': ds.SeriesInstanceUID,
            'InstanceNumber': ds.InstanceNumber,
            'SeriesNumber': ds.SeriesNumber,
        }
        info_list.append(info_dict)

    # 根据InstanceUID排序
    sort_list = sorted(info_list, key=lambda x: x['MediaStorageSOPInstanceUID'])

    return sort_list[-1]


# 将pdf报告转换后的图片数据写入DICOM影像，并放到原始DICOM文件夹中
def write_dcm(index, img, dicom, dicom_dir):
    # 用于将InstanceUID尾数加1
    chguid = lambda x: x.replace(x[-1], str(int(x[-1]) + index + 1))

    # 创建临时文件名
    suffix = '.dcm'
    filename_little_endian = tempfile.NamedTemporaryFile(suffix=suffix).name
    # filename_big_endian = tempfile.NamedTemporaryFile(suffix=suffix).name

    # 填充DICOM文件头必要信息
    file_meta = Dataset()
    file_meta.MediaStorageSOPClassUID = dicom['MediaStorageSOPClassUID']
    file_meta.MediaStorageSOPInstanceUID = chguid(dicom['MediaStorageSOPInstanceUID'])
    file_meta.ImplementationClassUID = dicom['ImplementationClassUID']

    # 创建一个FileDataset实例，初始时没有影像数据，只提供文件头(file meta)
    ds = FileDataset(filename_little_endian, {}, file_meta=file_meta, preamble=b"\0" * 128)

    # 加入DICOM tag信息（不包含所有）
    ds.PatientName = dicom['PatientName']
    ds.PatientID = dicom['PatientID']
    ds.PatientSex = dicom['PatientSex']
    ds.PatientBirthDate = dicom['PatientBirthDate']
    ds.PatientAge = dicom['PatientAge']

    # Set the transfer syntax
    # ds.is_little_endian = True
    # ds.is_implicit_VR = True

    # 加入DICOM tag信息（不包含所有）
    dt = datetime.datetime.now()
    ds.ContentDate = dt
    time_str = dt.strftime('%H%M%S.%f')
    ds.ContentTime = time_str

    # 生成StudyUID, SeriesUID, InstanceUID
    ds.SOPClassUID = dicom['SOPClassUID']
    ds.SOPInstanceUID = chguid(dicom['SOPInstanceUID'])
    ds.StudyInstanceUID = dicom['StudyInstanceUID']
    ds.SeriesInstanceUID = dicom['SeriesInstanceUID'] + '.1'
    ds.InstanceNumber = index + 1
    ds.SeriesNumber = str(int(dicom['SeriesNumber']) + 1)

    # 指定影像参数
    ds.Rows = img.size[1]
    ds.Columns = img.size[0]
    ds.SamplesPerPixel = 3
    ds.PhotometricInterpretation = "RGB"
    ds.HighBit = 7
    ds.BitsStored = 8
    ds.BitsAllocated = 8
    ds.PixelRepresentation = 0

    # 将img对象转换为bytes类型，写入PixelData字段
    ds.PixelData = img.tobytes()

    # 指定TransferSyntax等信息
    ds.file_meta.TransferSyntaxUID = pydicom.uid.ExplicitVRBigEndian
    ds.is_little_endian = False
    ds.is_implicit_VR = False

    # 保存影像文件
    file_name = 'report_{}.dcm'.format(index)
    file_path = os.path.join(dicom_dir, file_name)
    ds.save_as(file_path)


def main(pdf_path, dicom_dir):
    # 将pdf转换为图片
    images_list = convert_from_path(pdf_path)

    # 从DICOM文件夹中读取最后一张影像的信息
    dicom_info = read_dcm(dicom_dir)

    # 创建DICOM影像并写入图片数据
    for index, img in enumerate(images_list):
        write_dcm(index, img, dicom_info, dicom_dir)


if __name__ == '__main__':
    pdf_path = '/home/ray/dcm_report/report.pdf'
    dicom_dir = '/home/ray/dcm_report/dcm_data'

    main(pdf_path, dicom_dir)

```



