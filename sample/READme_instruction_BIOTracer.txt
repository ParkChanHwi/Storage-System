/*This is a readme file to introduce how to use the source file as well as the the steps to build and setup BIOTracer on a nexus 5.*/
/*** to refer this paper, please using the following info.

@inproceedings{zhou2015characteristics,
  title={I/O Characteristics of Smartphone Applications and Their Implications for eMMC Design},
  author={Zhou, Deng and Pan, Wen and Wang, Wei and Xie, Tao},
  booktitle={Workload Characterization (IISWC), 2015 IEEE International Symposium on},
  pages={12--21},
  year={2015},
  organization={IEEE}
}

***/

*****************************************************************************************************************************************************************************
Part 1: 
in case you want to diectly use the traces:
log files that used on our paper are included in the foler "trace_files" the format of the log is:
	column 0. acess start address in sectors 

	column 1. access size in sectors   note size should be time of 8 because basic block size is 4 KB or 8 sector. But mmc driver add addition sector to some of the size and thus you need to clean it.

	column 2. access size in byte.
	
	column 3. access type & waiting or not: the lowest bit to indicate read or write (0 is read and 1 is write), the third bit represent the request have be waiting or not (4 indicate no wait while 0 indicate has been waiting some time)
		For instance 5 represents that the request is a write request and the request does not wait before been processed, which indicates the queue is empty when the request comes.
	
	column 4. request generate time (the request been generated and inserted into request queue).

	column 5. request process start time (the request been fetched and dequeue by mmc driver from request queue and begin to process)

	column 6. time of request been submitted to hardware (the time that the driver issue the request to the hardware)

	column 7. request finish time (time of the callback function been invoked after request completion)

*****************************************************************************************************************************************************************************
Part 2:
Instruction for those who are interested in customer Android kernel compiling and installation on Nexus 5.
*** this instruction is wrote for Nexus 5 with android version 4.4. Uncompatible issue may need to be fixed if apply to other device.
##################################################################################################################################
1. requirement.
	a.	unlock Nexus 5. (there is lots of unlock tutorial online for unlock a Nexus 5).

	b.	Nexus 5 must be running with android 4.4 (android 5.0 or 5.1 maybe also ok since they are still use kernel version 3.4, but I didn't test it).

	c.	Linux enivonement for kernel compiling. (I used the ubuntu 14.04 with kernel version 3.14, Android kernel compiling is independent of host machine)

	d. 	ADB and fastboot tools: For ubuntu, they can be installed with command: # sudo apt-get install android-tools-adb android-tools-fastboot. guid can be found in link :http://bernaerts.dyndns.org/linux/74-ubuntu/328-ubuntu-trusty-android-adb-fastboot-qtadb

	e. 	Android kernel build enivonment setup: please follow the guid: https://source.android.com/source/building-kernels.html 
		Please see details about nexus5 kernel build from the link: http://marcin.jabrzyk.eu/posts/2014/05/building-and-booting-nexus-5-kernel.
		$ git clone https://android.googlesource.com/kernel/msm.git
		$ cd msm
		$ git branch -a
		$ git checkout origin/android-msm-hammerhead-3.4-kitkat-mr1
		*Note: the version we used during develop the BIOTracer is the android-msm-hammerhead-3.4-kitkat. You can always use a different version but may need extra efffort to fix the compatibility issue.
		1. Actually you don't need the full android source code tree. you can skip it and just setup kernel compiling enivonment. On a Linux host, if you don't have an Android source tree, you can download the prebuilt toolchain from:
			$ git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/arm/arm-eabi-4.6

			Note that both arm-eabi-4.6 and arm-eabi-4.8 works. if you encountered error during debugging, you can try to use the other one to bypass the error. 

	f. boot.img tools;
		In our source code folders there are two folders bootimg-tools and mkbootimg. 
			in /bootimg-tools/mkbootimg/ there is a executable program: unmbootimg, this command will be used to unpack the original boot.img to obtain the cprosspanding parameters and ramdisk.
			in /mkbootimg there is a executable program: mkbootimg, this command will be used to pack your own kernel image into a boot.img.
			*** note: the unmbootimg and mkbootimg come from different developers.  both of their original code has problem on either unpack or pack. So I just select the right one to use. You can add this two executable program into your path for convenivence. If encoured error on different platforms, you need to recompile them.

##################################################################################################################################
2. Kernel compiling.
	It's better to compile the original code without any modification for the first time. After first success, you can play on the code and do anything you are interested.
	a. Enivonment setup: Follow the command to start your first compiling
		$ export ARCH=arm
		$ export SUBARCH=arm
		$ export CROSS_COMPILE=arm-eabi-
		*note: you can add the export statements into you own .bashrc file so that don't need to redo it after each restart.
	b. Enter the root of your kernel source code. The msm project has the sources for ADP1, ADP2, Nexus One, Nexus 4, Nexus 5, Nexus 6, and can be used as a starting point for work on Qualcomm MSM chipsets.
		$ cd msm     

	c. searhing online for the basic usage of git. super easy.
		$ git checkout <commit_from_first_step> 

	d. Configure the kernel you want to compile.
		$ make hammerhead_defconfig

	e. Start compiling/building.
		$ make
##################################################################################################################################
3. boot.img build.
	a. After success on the step of 2.e, you should have a compilied kernal image under /msm/arch/arm/boot/. There are two version of kernel image one is namede "image" and the other one is "zImage-dtb". For nexus 5, we need the "zImage-dtb"
	b. obtaina factory image of Nexus 5. from https://developers.google.com/android/nexus/images?hl=en
	c. decompressing it and find the original boot.img in /hammerhead-kot49h/image-hammerhead-kot49h. 	*note: the "kot49h" can be different based on the version you are using.
	d. unpack the original boot.img with command 
		#unmkbootimg -i boot.img
		*** you will obtain the a packing command:
		$ unmkbootimg -i boot.img
		kernel written to 'kernel' (8479560 bytes)
		ramdisk written to 'ramdisk.cpio.gz' (660592 bytes)

		To rebuild this boot image, you can use the command:
  mkbootimg --base 0 --pagesize 2048 --kernel_offset 0x00008000 --ramdisk_offset 0x02900000 --second_offset 0x00f00000 --tags_offset 0x02700000 --cmdline 'console=ttyHSL0,115200,n8 androidboot.hardware=hammerhead user_debug=31 maxcpus=2 msm_watchdog_v2.enable=1' --kernel kernel --ramdisk ramdisk.cpio.gz -o factory_boot.img


	e. re-pack the boot.img with command:
		mkbootimg --base 0 --pagesize 2048 --kernel_offset 0x00008000 --ramdisk_offset 0x02900000 --second_offset 0x00f00000 --tags_offset 0x02700000 --cmdline 'console=ttyHSL0,115200,n8 androidboot.hardware=hammerhead user_debug=31 maxcpus=2 msm_watchdog_v2.enable=1' --kernel zImage-dtb --ramdisk ramdisk.cpio.gz -o myboot.img
		***The option -- kernel zImage-dtb indicate the kernel image you want to use in this new boot.img
		***Note: you can change the name "myboot.img" to any format like "xxx.img".
##################################################################################################################################
4. Install boot.img	details can be found in the link: http://forum.xda-developers.com/showthread.php?t=1752270
Using a USB cable to connect the nexus 5 to host.
	a. using adb devices to check the available device connected to the host.
		$ adb devices
	
	b. using following command to restart the device into fastboot mode.
		$ adb reboot bootloader	
	
	c. make sure the device is under fastboot mode. And then use follow command to flush your own image into the nexus 5. 
		$ fastboot flash boot myboot.img
Done ! you have sucessfully compiled and installed your own kernel into the device. restart the device and then you can find the version tag is different in the setting-> about the phone-> software-> kernel version.
##################################################################################################################################
5. Applying BIOTracer into kernel.

In downloaded folder, there is a subfolder named "msm_modified files".
Please replace the files in this folder into the kernel source code tree"msm" in corresponding locations.
then repeat the steps through 2, 3 ,4 and then you should have a customer kernel installed in the nexus 5.

Note that the location of the log file is "/dev/dzhou_test.txt" you should be able to access it through USB with ADB command.
In addition, the log file will be clean and reset on each restart as the file is located on a virtual block device instead of eMMC.
	
##################################################################################################################################
6. The Design of BIOTracer is introduced in our paper "I/O Characteristics of Smartphone Applications and Their Implications for eMMC Design"
	The implementation of the BIOTracer: Please refer to the paper for the figure.

Our I/O Monitor is implemented based on kernel 3.4.0, which is the default kernel version in the factory image of Nexus 5 (build number KOT49H). The major part of our code is located in the eMMC driver, which handles and optimizes requests. eMMC driver has a thread to fetch and process incoming requests from block layer. This thread continuously fetches requests from the request queue and then transforms them to eMMC requests, which will be sent to the hardware layer.

In the implementation, the following files are modifed: one file in block layer: /include/linux/blkdev.h, one file in kernel header: /include/linux/mmc/core.h, three files in the mmc driver: /drivers/mmc/card/queue.h, queue.c and /drivers/mmc/ block.c. In blkdev.h, a variable (start time) is added to record the request start time (obtained by the getnstimeofday()). Once the request comes to the eMMC layer, the start time is written to the record time structure in current mmc request, which is defined in the file core.h and initialized as record structure in main memory. To enable the system to record the start time, a macro is implemented to call the getnstimeofday() function and integrated to the set start timens() defined by the kernel. The record structure, which is defined in the queue.h and initialized together with the eMMC request thread, is used to keep request information including start time, service start time, finish time, direction, size, status. 

The information of logging buffer like logging buffer address, size, free space of logging buffer, logging file path/name is kept in record structure as well. The file queue.c includes the record buffer and writer implementation, the default buffer size is set as 32KB which can store record of over 300 requests. The start time of request is stamped when the requests are created by the file system and inserted into the request queue code within the blkdev.h. The eMMC request start time and the start address are set as creating time and start address of the first block request in the packing list because the eMMC driver
only fetches the following request and try to pack them.

The length of the eMMC request is set as the summary of length of the request within the pack list. The sevice start time is stamped when the packed eMMC request is issued to hardware (see step 2 in Figure 2). Then in the block.c the return time is obtained when the request is completed (step 3 in Figure 2). In order to understand the I/O pattern better, we also record the no-waiting status of the requests. The no-waiting status of the request is checked and set by eMMC driver, the truth value indicates that there is no undergoing request when this request comes. This function is implemented in the file block.c. The reason why we don’t check this status at the block layer is because that the request can stay a long time in eMMC driver layer for request transforming and packing, which may cost long time to process and makes the recording not accurate.




