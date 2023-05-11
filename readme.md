## 将FARO的扫描文件转换为LAS/LAZ文件

## 环境配置

+ 安装FARO SDK文件，版本号为：1.1.905.1
双击**[FARO_LS_SDK_2021.5.1.9021_Setup.exe](https://downloads.faro.com/index.php/s/9i8L7HwCWNfSyyF)**文件进行安装
+ 注册SDK dll文件
双击**注册（以管理员身份运行）.bat**即可

## 转换示例

```bat
readFls.exe *.fls *.laz/*.las -10 10 -10 10

```

## 主要代码


```c++

#include <iostream>

#include <cstring>
#include <string>

#include <time.h>

#include "lasreader.hpp"
#include "laswriter.hpp"

#ifdef _WIN64
// Yes - type is 'win32' even on WIN64!
#pragma comment(linker, "\"/manifestdependency:type='win32' name='FARO.LS' version='1.1.905.1' processorArchitecture='amd64' publicKeyToken='1d23f5635ba800ab'\"")
#else
#pragma comment(linker, "\"/manifestdependency:type='win32' name='FARO.LS' version='1.1.905.1' processorArchitecture='amd64' publicKeyToken='1d23f5635ba800ab'\"")
#endif

// These imports just defines the types - they don't determine which DLLs are actually loaded!
// You can choose whatever version you have installed on your system - as long as the interface is compatible
#import "C:\Windows\WinSxS\amd64_faro.ls_1d23f5635ba800ab_1.1.905.1_none_36a3d9b167a7571f\iQOpen.dll" no_namespace
#import "C:\Windows\WinSxS\amd64_faro.ls_1d23f5635ba800ab_1.1.905.1_none_36a3d9b167a7571f\FARO.LS.SDK.dll" no_namespace


int main(int argc, char** argv)
{
	if (argc != 7)  {
		std::cerr << " need 3 parameters: " << argv[0] << " *.fls *.laz/*.las minx maxx minz maxz" << std::endl;
		return -1;
	}
	bstr_t fls_path = argv[1];
	std::string outp = argv[2];

	const double ext_minx = atof(argv[3]);
	const double ext_maxx = atof(argv[4]);
	const double ext_minz = atof(argv[5]);
	const double ext_maxz = atof(argv[6]);

	if (ext_minx >= ext_maxx) {
		std::cerr << "minx > maxx" << std::endl;
		return -1;
	}
	if (ext_minz >= ext_maxz){
		std::cerr << "minz > maxz" << std::endl;
		return -1;
	}
	//std::cout << printf("\tminx=%f,maxx=%f,minz=%f,maxz=%f\n", ext_minx, ext_maxx, ext_minz, ext_maxz) << std::endl;

	CoInitialize(NULL);   //应用程序调用com库函数（除CoGetMalloc和内存分配函数）之前必须初始化com库
	// FARO LS Licensing
	// Please note: The cryptic key is only a part of the complete license
	// string you need to unlock the interface. Please
	// follow the instructions in the 2nd line below

	BSTR licenseCode = L"FARO Open Runtime License\n"
		//#include "../FAROOpenLicense"    // Delete this line, uncomment the following line, and enter your own license key here:
		L"Key:*************************\n"
		L"\n"
		L"The software is the registered property of FARO Scanner Production GmbH, Stuttgart, Germany.\n"
		L"All rights reserved.\n"
		L"This software may only be used with written permission of FARO Scanner Production GmbH, Stuttgart, Germany.";

	clock_t start, t1, t2, end;
	start = clock();
	IiQLicensedInterfaceIfPtr liPtr(__uuidof(iQLibIf));
	liPtr->License = licenseCode;
	IiQLibIfPtr libRef = static_cast<IiQLibIfPtr>(liPtr);
	if (libRef->load(fls_path) != 0)   //读取文件的全路径  切记
	{
		std::cout << "load  ScanData errer !!!" << std::endl;
		libRef = NULL;
		liPtr = NULL;
		return -1;
	}
	std::cout << "> start to read: " << argv[1] << std::endl;

	libRef->scanReflectionMode = 2;            //黑白灰度展示图像
	//std::cout << libRef->scanReflectionMode << std::endl;
	//libRef->scanReflectionMode = 1;    //默认为1  可以为0 1 2三个模式。

	int ret = libRef->setAttribute("#app/ScanLoadColor", "2");    //设置为彩色 Load grey information instead of color
	//std::cout << "setAttribute: " << ret << std::endl;

	double x, y, z;
	double rx, ry, rz, angle;
	libRef->getScanPosition(0, &x, &y, &z);
	libRef->getScanOrientation(0, &rx, &ry, &rz, &angle);
	int numRows = libRef->getScanNumRows(0);
	int numCols = libRef->getScanNumCols(0);

	t1 = clock();

	std::cout << "\tnumRows: " << numRows << std::endl;
	std::cout << "\tnumCols: " << numCols << std::endl;
	//std::cout << x << "," << y << "," << z << std::endl;
	//std::cout << rx << "," << ry << "," << rz << "," << angle << std::endl;
	std::cout << "> 初始化文件耗时：" << (t1 - start) / CLOCKS_PER_SEC << " Secs" << std::endl;
	std::cout << "\tnumber of points: " << numCols * numRows << std::endl;

	LASwriteOpener laswriterOpener;
	laswriterOpener.set_file_name(outp.c_str());
	std::string suffix = outp.substr(outp.find_last_of('.') + 1);
	if (strcmp(suffix.c_str(), "laz") == 0 || strcmp(suffix.c_str(), "LAZ") == 0) {
		std::cout << "> output format: laz" << std::endl;
		laswriterOpener.set_format("laz");
	}
	LASheader lasheader;
	lasheader.x_offset = .0;
	lasheader.y_offset = .0;
	lasheader.z_offset = .0;
	/*
	* point_data_format:
	* 0:las 1.2; 1:las 1.3; 2:las 1.4; 3: no gps time; 4: no rgb; 5: no near red band; 6: no know band; 7: laz format
	*/
	lasheader.point_data_format = 1;
	// las 1.2 val=28; others val=34;
	lasheader.point_data_record_length = 34;
	lasheader.number_of_point_records = numCols * numRows;
	LASpoint laspoint;
	laspoint.init(&lasheader, lasheader.point_data_format, lasheader.point_data_record_length, 0);
	LASwriter *laswriter = laswriterOpener.open(&lasheader);
	if (laswriter == 0){
		fprintf(stderr, "\nErr: could not open laswriter!\n");
		return false;
	}
	double minx = .0, miny = .0, minz = .0;
	double maxx = 0.0, maxy = .0, maxz = .0;
	double pt_x, pt_y, pt_h;
	int intensity;
	double gps_time;

	//double x, y, z;
	int refl;
	// Access all points points by point
	long counter = 0;
	for (int col = 0; col < numCols; col++)
	{
		for (int row = 0; row<numRows; row++)
		{
			int result = libRef->getScanPoint(0, row, col, &x, &y, &z, &refl);
			//Use getXYZScanPoints or getXYZScanPoints2 instead to read multiple scan points in a single call. For example, you can read the scan points column by column with these two methods.

			pt_x = x;
			//pt_y = y;
			// extend the y coordination to line
			pt_y = (col < numCols / 2 ? col * 0.005 : (col - numCols / 2) * 0.005);
			gps_time = (col < numCols / 2 ? col * 0.01: (col - numCols / 2) * 0.01);
			pt_h = z;
			intensity = refl;

			if (pt_x < ext_minx || pt_x > ext_maxx) continue;
			if (pt_h < ext_minz || pt_h > ext_maxz) continue;

			counter++;

			laspoint.set_x(pt_x);
			laspoint.set_y(pt_y);
			laspoint.set_z(pt_h);
			laspoint.set_intensity(intensity);
			laspoint.set_gps_time(gps_time);

			minx = pt_x < minx ? pt_x : minx;
			miny = pt_y < miny ? pt_y : miny;
			minz = pt_h < minz ? pt_h : minz;

			maxx = pt_x > maxx ? pt_x : maxx;
			maxy = pt_y > maxy ? pt_y : maxy;
			maxz = pt_h > maxz ? pt_h : maxz;

			laswriter->write_point(&laspoint);
			laswriter->update_inventory(&laspoint);


		}
	}

	lasheader.number_of_point_records = counter;
	std::cout << "\tlas writer info: " << std::endl;
	std::cout << "\tnumber_of_point_records=" << lasheader.number_of_point_records << std::endl;
	lasheader.set_bounding_box(minx, miny, minz, maxx, maxy, maxz);
	laswriter->update_header(&lasheader);
	laswriter->close();
	delete laswriter;

	libRef = NULL; liPtr = NULL;
	CoUninitialize();

	t2 = clock();
	std::cout << "> 处理耗时：" << (t2 - t1) / CLOCKS_PER_SEC << " Secs" << std::endl;

	return 0;
}

```

## 特别鸣谢

+ Martin Isenburg
+ FUCKING FARO SDK

## 参考

+ https://www.cnblogs.com/lovebay/p/9337565.html
+ https://github.com/LAStools/LAStools
+ https://knowledge.faro.com/Software/FARO_SCENE/SCENE/Download_and_Installation_of_the_FARO_LS_SDK
