# AvxToNeon

#### Introduction
A system acceleration library for porting from x86 architecture to Kunpeng architecture.
When applications using Intel intrinsic instructions are ported from the x86 architecture to Kunpeng architecture, the instructions need to be further developed because the Arm64 instruction names and functions are different from that of x86. As a result, huge porting workloads are caused. In this project, the frequently used AVX instructions are encapsulated as independent modules to reduce repeated development workload. The AVX instructions are replaced with related NEON SIMD instructions, while the instruction names and functions remain unchanged. Users can replace the corresponding instructions by importing related header files into the application software. 

#### Architecture
- data: folder of test data
- tests: folder of test source code
- avx2neon.h: external header file 
- avx512intrin.h: internal header file 
- avxintrin.h: internal header file 
- emmintrin.h: internal header file 
- immintrin.h: internal header file 
- typedefs.h: internal header file 
- supportedlist.md: list of supported interfaces

#### Instructions

1.  Get AvxToNeon source code
2.  Shield immintrin.h
3.  Include avx2neon.h in your application
4.  Add compilation options, ARCH_CFLAGS = -march=armv8-a+fp+simd+crc

#### License

It is licensed under the [APACHE LICENSE, VERSION 2.0](https://www.apache.org/licenses/LICENSE-2.0). For more information, see the license file.. 

#### Test

This project also provides interface test cases for developers. The logic implementation code of test cases is located in the tests directory, and the input data and expected output of the test cases are in the data directory. Use the following commands to perform test cases:

```
(1) cd tests
(2) make
(3) ./test
```

After the **test** command is executed, information similar to the following is displayed on the console:

```
Running Test MM512_CASTPS128_PS512

...

Running Test MM256_SET_EPI32

AVX2NEONTest Complete: Passed 265 tests: Failed 0
```

All the instructions provided in this project have been verified on OpenEuler 20.03 LTS SP1 and CentOS Linux release 7.6.1810 (AltArch), and GCC 7.3, GCC 4.8.5, and GCC 9.2.0.

#### Details

For more information, you can visit

<https://www.huaweicloud.com/kunpeng/software.html>

If you have questions or comments, you can create an issue on gitee. If you want to contact the developers team directly, you can send email to

 [kunpengcompute@huawei.com](mailto:kunpengcompute@huawei.com).

#### Code

If you want to get source code of functions listed in supportedlist.md, you can send an email to [kunpengcompute@huawei.com](mailto:kunpengcompute@huawei.com).

#### Contribution

1.  Fork the repository
2.  Create Feat_xxx branch
3.  Commit your code
4.  Create Pull Request

#### Copyright

Copyright © 2020 Huawei Corporation. All rights reserved. 