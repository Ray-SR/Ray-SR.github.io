---
title: DICOM Tag中文编码问题
date: 2019-10-15 17:38:49
tags: [DICOM,Python,pydicom]
categories: DICOM
---

DICOM Tag信息一般都以英文存储，出现中文时，可能会出现显示为乱码的情况，而导致乱码的原因通常是，存储的信息含有中文，而DICOM tag中指定的编码格式`SpecificCharacterSet`并不支持中文.

<!--more-->

## 编码问题

常见的DICOM tag编码格式一般是`ISO_IR 100`，存储于`SpecificCharacterSet`（即0008,0005） tag中，该编码格式并不支持中文，因此DICOM信息中出现中文时，最好将这个tag的值改为`ISO_IR 192`即UTF-8编码。常见的DICOM tag编码对应关系可参考：

```python
	'ISO_IR 100': 'latin_1',
    'ISO_IR 101': 'iso8859_2',
    'ISO_IR 109': 'iso8859_3',
    'ISO_IR 110': 'iso8859_4',
    'ISO_IR 126': 'iso_ir_126',  # Greek
    'ISO_IR 127': 'iso_ir_127',  # Arabic
    'ISO_IR 138': 'iso_ir_138',  # Hebrew
    'ISO_IR 144': 'iso_ir_144',  # Russian
    'ISO_IR 148': 'iso_ir_148',  # Turkish
    'ISO_IR 166': 'iso_ir_166',  # Thai
    'ISO 2022 IR 6': 'iso8859',  # alias for latin_1 too
    'ISO 2022 IR 13': 'shift_jis',
    'ISO 2022 IR 87': 'iso2022_jp',
    'ISO 2022 IR 100': 'latin_1',
    'ISO 2022 IR 101': 'iso8859_2',
    'ISO 2022 IR 109': 'iso8859_3',
    'ISO 2022 IR 110': 'iso8859_4',
    'ISO 2022 IR 126': 'iso_ir_126',
    'ISO 2022 IR 127': 'iso_ir_127',
    'ISO 2022 IR 138': 'iso_ir_138',
    'ISO 2022 IR 144': 'iso_ir_144',
    'ISO 2022 IR 148': 'iso_ir_148',
    'ISO 2022 IR 149': 'euc_kr',
    'ISO 2022 IR 159': 'iso2022_jp_2',
    'ISO 2022 IR 166': 'iso_ir_166',
    'ISO 2022 IR 58': 'iso_ir_58',
    'ISO_IR 192': 'UTF8',  # from Chinese example, 2008 PS3.5 Annex J p1-4
    'GB18030': 'GB18030',
    'ISO 2022 GBK': 'GBK',  # from DICOM correction CP1234
    'ISO 2022 58': 'GB2312',  # from DICOM correction CP1234
    'GBK': 'GBK',  # from DICOM correction CP1234
```

## 使用SimpleITK读取DICOM tag
DICOM tag中含有中文时，可以利用SimpleITK读取Tag信息并转换为GBK编码，随后可以使用pydicom改变`SpecificCharacterSet`的值为`ISO_IR 192`(暂时未找到SimpleITK改变DICOM tag的方法)。以下是读取DICOM tag中`PatientName`并改变编码的示例：
```python
import SimpleITK as sitk

dcm_path = '/path_to_dcm'
dcm = sitk.ReadImage(dcm_path)
patient_name = dcm.GetMetaData('0010|0010').strip().encode("utf-8", "surrogateescape").decode('gbk', 'replace')

ds = pydicom.dcmread(dcm_path)
ds.SpecificCharacterSet = 'ISO_IR 192'
ds.PatientName = patient_name
ds.save_as(dcm_path)

```

