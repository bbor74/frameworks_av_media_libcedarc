1. linux related instructions:

1. The compilation tool chain corresponding to each set of linux so is as follows：

# arm926-uclibc for BU1 C500
# arm-openwrt-linux-muslgnueabi for BU2 Tina2.0
# arm-openwrt-linux-muslgnueabi-v for v5
# arm-openwrt-linux-uclibc for BU2 R16
so_dir: arm926-uclibc                 host: arm-linux              toolchain_dir: uclibc_arm926_static
so_dir: arm-linux-gnueabi             host: arm-linux-gnueabi      toolchain_dir: external-toolchain
so_dir: arm-linux-gnueabihf host: arm-linux-gnueabihf toolchain_dir: The system comes with
so_dir: arm-none-linux-gnueabi host: arm-none-linux-gnueabi toolchain_dir: The system comes with
so_dir: arm-openwrt-linux-muslgnueabi host: arm-openwrt-linux      toolchain_dir: OpenWrt-Toolchain-sunxi_gcc-5.2.0_musl-1.1.12_eabi.Linux-x86_64
so_dir: arm-openwrt-linux-uclibc      host: arm-openwrt-linux      toolchain_dir: toolchain_arm_uClibc
so_dir: arm-linux-gnueabihf-linaro    host: arm-linux-gnueabihf    toolchain_dir: gcc-linaro-arm-linux-gnueabihf-4.8-2014.01_linux
so_dir: arm-aarch64-openwrt-linux     host: aarch64-openwrt-linux  toolchain_dir: toolchain-sunxi-tina2.0-64
so_dir: arm-openwrt-linux-muslgnueabi-v5  host: arm-openwrt-linux-muslgnueabi    toolchain_dir: linux-x86

2.The compilation steps are as follows (take arm926-uclbc as an example)：

2.1 export compilation tool chain：
                     TOOLS_CHAIN=/home/user/workspace/tools_chain/
                     export PATH=${TOOLS_CHAIN}/uclibc_arm926_static/bin:$PATH

2.2. Related tools for running automake:./bootstrap

2.3. Configure makefile：
2.3.1 mode:./configure --prefix=INSTALL_PATH --host=HOST_NAME LDFLAGS="-LSO_PATH"
2.3.2 Example:./configure --prefix=/home/user/workspace/libcedarc/install --host=arm-linux LDFLAGS="-L/home/user/workspace/libcedarc/lib/arm926-uclibc"
2.3.3 Special note: if the kernel uses linux3.10, you must add a flag: CFLAGS="-DCONF_KERNEL_VERSION_3_10" CPPFLAGS="-DCONF_KERNEL_VERSION_3_10"
      Namely:./configure --prefix=/home/user/workspace/libcedarc/install --host=arm-linux CFLAGS="-DCONF_KERNEL_VERSION_3_10" CPPFLAGS="-DCONF_KERNEL_VERSION_3_10" LDFLAGS="-L/home/user/workspace/libcedarc/lib/arm926-uclibc"

2.4 Compilation: make; make install

2. Description of version changes：

1). CedarC-v1.0.4

1. Special note：

1.1 Camera module related: (>= android7.0 platform needs to be modified as follows, other platforms maintain the previous practice)
    Reason for modification: a.Neither omx nor the Android framework subdivides the two image formats NV21 and NV12, both of which use OMX_COLOR_FormatYUV420SemiPlanar.
                For representation, OMX_COLOR_FormatYUV420SemiPlanar can represent NV21 or NV12.；
                b.While ACodec uses OMX_COLOR_FormatYUV420SemiPlanar for NV12, camera uses OMX_COLOR_FormatYUV420SemiPlanar for NV21，
                The previous practice was that omx_venc distinguished compatibility by the package name of the process. If the caller is ACodec, it is used as NV12, and if the caller is camera, it is used as NV12.
                NV21；
                c.android7.0 cannot obtain the package name of the process due to rights management reasons, so it cannot be compatible in omx_venc, and can only be used in the upper caller layer.
                Compatible, and cts will test the interface of ACodec. If you change ACodec, it will respond to the cts test, so you can only modify the camera.
                d.The principle of modification is: the member variable OMX_COLOR_FormatYVU420SemiPlanar of the enumeration type of the extended image format is used to represent NV21；
                  That is, OMX_COLOR_FormatYUV420SemiPlanar--> NV12
                     OMX_COLOR_FormatYVU420SemiPlanar --> NV21

    Modified place：
    a. Synchronization header file: synchronize openmax/omxcore/inc/OMX_IVCommon.h to./native/include/media/openmax/OMX_IVCommon.h
    b. The camera module needs to be modified when calling the openmax/venc encoding interface：
       NV21 before modification --> OMX_COLOR_FormatYUV420SemiPlanar，
       Modified NV21 --> OMX_COLOR_FormatYVU420SemiPlanar;

1.2 Instructions for loading the decoding library：
    The previous operation of loading the decoding library is in vdecoder.The c layer is carried out, in order to improve flexibility, from v1.0.4 onwards
Put the operation of loading the decoding library to the upper caller, and the upper caller can call vdecoder.c's AddVDPlugin interface defaults
Load all sub-decoders, or refer to the following code to load on demand：

static void InitVDLib(const char *lib)
{
    void *libFd = NULL;
    if(lib == NULL)
    {
        loge(" open lib == NULL ");
        return;
    }

    libFd = dlopen(lib, RTLD_NOW);

    VDPluginFun *PluginInit = NULL;

    if (libFd == NULL)
    {
        loge("dlopen '%s' fail: %s", lib, dlerror());
        return ;
    }

    PluginInit = (VDPluginFun*)dlsym(libFd, "CedarPluginVDInit");
    if (PluginInit == NULL)
    {
        logw("Invalid plugin, CedarPluginVDInit not found.");
        return;
    }
    logd("vdecoder open lib: %s", lib);
    PluginInit(); /* init plugin */
    return ;
}

static void AddVDLib(void)
{
    InitVDLib("/system/lib/libawh265.so");
    InitVDLib("/system/lib/libawh265soft.so");
    InitVDLib("/system/lib/libawh264.so");
    InitVDLib("/system/lib/libawmjpeg.so");
    InitVDLib("/system/lib/libawmjpegplus.so");
    InitVDLib("/system/lib/libawmpeg2.so");
    InitVDLib("/system/lib/libawmpeg4base.so");
    InitVDLib("/system/lib/libawmpeg4normal.so");
    InitVDLib("/system/lib/libawmpeg4vp6.so");
    InitVDLib("/system/lib/libawmpeg4dx.so");
    InitVDLib("/system/lib/libawmpeg4h263.so");
    InitVDLib("/system/lib/libawvp8.so");

    return;
}

2. The changes are as follows：
2.1 openmax:venc add p_skip interface
2.2 h265:fix the HevcDecodeNalSps and HevcInitialFBM
2.3 h264:refactor the H264ComputeOffset
2.4 Correction of mjpeg scale+rotate
2.5 vdcoder/h265: add the code of parser HDR info
2.6 vdecoder/h265: add the process of error-frame
2.7 vdecoder/h264: make sure pMbNeighborInfoBuf is 16K-align to fix mbaff function
2.8 openmax/vdec: remove cts-flag
2.9 openmax/venc: remove cts-flag
2.10 detection a complete frame bitstream and crop the stuff zero data
2.11 vdecoder/h265:fix the bug that the pts of keyFrame is error when seek
2.12 openmax/inc: adjust the define of struct
2.13 vencoder: add lock for vencoderOpen()
2.14 vdecoder/ALMOST decoders:fix rotate and scaledown
2.15 vdecoder/h265:fix the process of searching the start-code when sbm cycles
2.16 vdecoder/h265:fix the bug when request fbm fail after fbm initial
2.17 vdecoder/h265:improve the process when poc is abnormal
2.18 cedarc: unify the release of android and linux
2.19 vdecoder/avs: make mbInfoBuf to 16K align

2).CedarC-v1.0.5

1. The changes are as follows：
1.1.configure.ac:fix the config for linux compiling
1.2.openmax/venc: revert mIsFromCts on less than android_7.0 platfrom
1.3.vdecoder/h265soft:make h265soft be compatible with AndroidN
1.4.cedarc: merge the submit of cedarc-release
1.5.vdecoder/h265:use the flag "bSkipBFrameIfDelay"
1.6.vdecoder:fix the buffer size for thumbnail mode
1.7.cedarc: fix compile error of linux
1.8.omx:venc add fence for androidN
1.9.openmax:fix some erros
1.10.omx_venc: add yvu420sp for omx_venc init
1.11.videoengine:add 2k limit for h2
1.12.cedarc: add the toolschain of arm-openwrt-linux-muslgnueabi for v5
1.13.cedarc: Solve the problem of 0x1663 mpeg4 playing flower screen
1.14.vdecoder: fix compile error of soft decoder for A83t
1.15.omx_vdec: fix the -1 bug for cts
1.16.Modify the way mpeg2 gets ve version
1.17.vencoder: fix for input addrPhy check
1.18.cedarc: merge the submit of cedarc-release

3). CedarC-v1.1

1. Special note：
1.1 v1.1 integrates the modified results of the H6-dev development branch; adds h265 10bit and afbc functions, this function requires
    cedarx matches the modification, otherwise it cannot take effect independently；
1.2 v1.1 closes the memory interface used to apply for physical continuous memory. If the upper layer needs to apply for physical memory, the module is required.
    Implement memory's interface internally or call other interfaces；
1.3 The two interfaces VideoDecoderGetVeIommuAddr and VideoDecoderFreeVeIommuAddr have been added for
    Bind and unbind the buffer of iommu.

2. The changes are as follows：
2.1 Added support for h265 10bit and afbc functions；
2.2 Add vp9 hardware decoding driver；
2.3 Added support for iommu buffer management；
2.4 Refactoring the ve module；
2.5 Refactoring the sbm module；
2.6 The memory interface is no longer open to the public.

4). CedarC-v1.1.1

1. The changes are as follows：
1.1 Adjust cedarc-release, the cedarc currently released can be applied to all Android and linxu platforms；
1.2 vdecoder/Vp8:process the case of showFrm
1.3 vdecoder/h265: increase the size of HEVC_LOCAL_SBM_BUF_SIZE
1.4 vdecoder/videoengine: add the function of checkAlignStride
1.5 vdecoder/h265: set proc info
1.6 vdecoder/sbm: Modify the bug that H265 sbmFrame access crosses the boundary

5). CedarC-v1.1.2

1. The changes are as follows：
1.1 vdecoder/h264: set proc info
1.2 vdecoder/mjpge: set proc info
1.3 vdecoder: improve function of savePicture
1.4 vencoder: fix for jpeg get phy_addr and androidN get chroma addr
1.5 Add offset information to fbmInfo
1.6 openmax/vdec: set mCropEnable to false on linux
1.7 Use attribute detection to determine how memory is used
1.8 unmap the fbm buffer when native window changed
1.9 vp8 return the alterframe error for'New crane needle.mkv'

6). CedarC-v1.1.3
1.1 vdecoder: change the 6k range
1.2 openmax:venc: fix for recorder
1.3 vdecoder: fix the bug for initializeVideoDecoder fails
1.4 vdecoder/h265Soft: fix the bug: crash when seek the video H265_test.mkv
1.5 ve: control phyOffset in ve module
1.6 use iomem.type to check iommu mode
1.7 vdecoder: add nBufFd when call FbmReturnReleasePicture
1.8 openmax: load the libawwmv3.so when init
1.9 vdecoder/VP9:reset some parameter for Vp9HwdecoderReset()
1.10 ve: dynamic set dram high channal

7). CedarC-v1.1.4
1.1 vencoder: add for thumb write back func
1.2 vencoder:fix for set thumb write back scaler dynamic
1.3 vencoder: fix for only thumb write back no encode
1.4 Modify the waiting time of the status bit of the detection hardware busy
1.5 vdecoder/sbmFrame: fix the error video 720P_V_.HEVC_AAC_2M_15F.mkv

8). CedarC-v1.1.6

1. The changes are as follows：
1.1 vdecoder/h264:after reset,the first frame pts is same to the last bitstream
1.2 h265:fix the bug of parse-extradata
1.3 Modify the bug of abnormal pts after H264 skip broadcast
1.4 vdecoder: add lock for VideoEngineCreate
1.5 fix gts test fail
1.6 vdecoder/vc1: fix the bug: error when seek
1.7 Adjust the H264 code architecture to clear the bug that the last frame was not decoded
1.8 openmax/vdec: not support metadata buffer
1.9 ve: fix for getIcVersion when other process is reseting ve
1.10 openmax/vdec: open mem-ops when use
1.11 vdecoder/h264/distinguish SbmReturnStream of stream and frame for resolution change
1.12 omx:venc: fix for recorder of h6-kk44-iommu
1.13 openmax/vdec: support afbc function
1.14 h264:fix the progress of erro
1.15 openmax/vdec: plus timeout to 5s

2.Special note:
2.1 mediacodec channel support for afbc function
    The support of afbc function involves two modules. One is the modification of the cedarc/openmax module. The modification of this module has been completed and will
 cedarc's code can be updated to cedarc-v1.1.6 or later；
    The second is the modification of the framework layer. At present, the patch of the framework layer is only integrated into the H6 scheme. If other schemes want to integrate afbc, the patch of the framework layer is only integrated into the H6 scheme.
 For the function, you can find Wang Xiwang of AL3 to provide framework_patch.
 
9). CedarC-v1.1.7

1. The changes are as follows：
1.1. vdecoder/avs:the case of diff pts > 2s for TvStream
1.2. vencoder:jpeg fix for exif buffer memory leak
1.3. h264:fix the bug of frameStream-end
1.4. vdecoder: fix the bug when get memops
1.5. vdecoder/h264:u16 to s32
1.6. vdecoder: add iptv-info for h264 and h265
1.7. vdecoder/fbm: avoid memory leak
1.8. Solve the problem of H8 Great White Shark. mp4 playing flower screen
1.9. vdecoder/sbmH264: surpport secure video
1.10. Apply for the size of the Dram buffer according to the resolution of H264
1.11. Fixed the playback of A026 when the surface is switched.mpeg4 stuck problem
1.12. Support iommu in secure mode
1.13. The ve frequency of H265 is adjusted to 696MHZ on H6
1.14. memory: fix for get phy_addr 0 when is iommu mode
1.15. H264 Add code for software header information
1.16. openmax/vdec: increase the input-buffer-size to 6 MB
1.17. vdecoder: optimize the policy of set vefreq
1.18. Add recognition of error frames in mpeg2 decoder
1.19. omx:venc: fix for gpu buf
1.20. vdecoder/avs/fix the flag of bIsProgressive
1.21. openmax/vdec: support native-handle
1.22. Add ve frequency configuration item
1.23. vdecoder: not get the veLock with the softDecoder case

10). CedarC-v1.1.7 -- patch-002

1. The changes are as follows：
1.1. Modify the way H264 calculates the dram buffer
1.2. Modify the decoding method of T3 mpeg4v2, this format VE does not support hard decoding
1.3. Compatible with the processing of extradata in widewine mode
1.4. vdecoder/H264/decoder one frame then return
1.5. Compatible with cases where parser transmits the wrong width and height
1.6. openmax/vdec: add the policy of LIMIT_STREAM_NUM
1.7. videoengine:fix the specificdat value
1.8. videoengine: fix the ve unlock in VideoEngineReopen
1.9. openmax/vdec: add the function of di
1.10. vdecoder/H264:resolution change for online video
1.11. Fixed a bug where the decoder received the eos mark and could not exit normally when the fbm information was not obtained.
1.12. openmax/vdec: remove bCalledByOmxFlag
1.13. h265: fix the bug decoding the slcieRps as numOfRefs is outoff range
1.14. openmax/vdec: not init decoder in the status of idle

10). CedarC-v1.1.7 -- patch-003

1. The changes are as follows：
1.1 Repair h264 cts failed
1.2 openmax/vdec: reset MAX_NUM_OUT_BUFFERS from 15 to 4 as it consume too much buffer
1.3 h265: fix the pts of eos frame for gts
1.4 fbm: fix the value of pMetaData
1.5 Modify mpeg2 pt2.vob crash bug

10). CedarC-v1.1.7 -- patch-004

1. The changes are as follows：
1.1 openmax/vdec: fix the process of decoding the last frame which size changes
1.2 h265: fix the bug of decoding extraData
1.3 fbm: add code for allocating metadata buffer for linux
1.4 di not support 4K stream, for 4K interlace stream, ve does scaledown
1.5 vdecoder: limit nVbvBufferSize to [1 MB, 32MB]
1.6 openmax/vdec: increase OMX_VIDEO_DEC_INPUT_BUFFER_SIZE_SECURE from 256 KB to 1 MB
1.7 Modify the problem of pts calculation error caused by frame loss in H264
1.8 vdecoder:add the decIpVersion for T7
1.9 vdecoder/h264:add reset parameters of H264ResetDecoderParams()
1.10 openMAX: Adapt DI process with two input di pictures to the platform of H3
1.11 openmax/vdec: just set nv21 format in di-function case
1.12 demo: add vencDem to cedarc
1.13 support the field structure of vc1 frame packed mode
1.14 demo:demoVencoder: fix for style error    
1.15 config: add config file of T7 platform

11). CedarC-v1.1.8

1. The changes are as follows：
1.1. Fixed a bug that could not be decoded when there was no specialdata in mp4normal thumbnail mode
1.2. Fixed the bug of mpeg4Normal format thumbnail
1.3. Support decoding of mjpeg444
1.4. vdecoder: fix the bug when sbm inits failed
1.5. vdecoder:fix for VC1. Be compatible to 64 bit system
1.6. Fix the problem of 0x1663 playing mpeg4 file flower screen
1.7. cdcUtil: fix ion handle for linux4.4
1.8. Restore the setting method of vbv buffer size
1.9. omx:venc: fix for h265 enc error
1.10. vdecoder:A63 upgrade,Mpeg1/2/4 addr register
1.11. vdecoder:catch DDR value for H265
1.12. Fixed playback 05_100M.ts video pts error problem
1.13. cedarc:avs:add support for sun8iw7p1
1.14. The mjpeg of the 1708 chip calls mjpegPlus
1.15. vdecoder/h265:protecting nFrameDuration
1.16. vdecoder/sbm: fix bug: crash when length of stream is 0
1.17. Add support for andoido
1.18. Modify the frame loss mechanism of H265
1.19. cedarc:avs_plus:fix avs_plus unsupport error on chip-1680
1.20. cedarc/log: dynamic show log by property_get
1.21. cedarc: compile so in system/lib/ and system/vendor/lib
1.22. vdecoder/fbm: reduce frmbuf_c_size of afbc 

11). CedarC-v1.1.9

1. The changes are as follows：

1.1 Modify the name of the linux directory library：

    Before modification after modification
arm-linux-gnueabi                    arm-linux-gnueabi
arm-linux-gnueabihf                  arm-linux-gnueabihf
arm-none-linux-gnueabi               arm-none-linux-gnueabi
arm-linux-gnueabihf-linaro           toolchain-sunxi-aarch64-glibc
arm926-uclibc                        toolchain-sunxi-aarch64-musl
arm-aarch64-openwrt-linux            toolchain-sunxi-arm9-glibc
arm-openwrt-linux-muslgnueabi        toolchain-sunxi-arm9-musl
arm-openwrt-linux-muslgnueabi-v5     toolchain-sunxi-arm-glibc
arm-openwrt-linux-uclibc             toolchain-sunxi-arm-musl

Description: The reason for the modification is: Organize according to the latest tool chain provided by BU1；
      The first three sets are more used compilation toolchains, which are suitable for the development of various BU programs.；
      The next 6 sets of SO compiled from the tool chain provided by BU1. If other BUS need to be used, it is recommended to communicate with colleagues in the cedarc driver group (such as Wang Xiwang/Liu Xiaoyan) to understand.；

1.2 Modify the linux so name：
 
    Before modification after modification
libcdc_vd_avs.so                   libawavs.so
libcdc_vd_h264.so                  libawh264.so
libcdc_vd_h265.so                  libawh265.so
libcdc_vd_mjpeg.so                 libawmjpeg.so
libcdc_vd_mjpegs.so                libawmjpegplus.so
libcdc_vd_mpeg2.so                 libawmpeg2.so
libcdc_vd_mpeg4base.so             libawmpeg4base.so
libcdc_vd_mpeg4dx.so               libawmpeg4dx.so
libcdc_vd_mpeg4h263.so             libawmpeg4h263.so
libcdc_vd_mpeg4normal.so           libawmpeg4normal.so
libcdc_vd_mpeg4vp6.so              libawmpeg4vp6.so
libcdc_vd_vp8.so                   libawvp8.so
libcdc_vd_vp9Hw.so                 libawvp9Hw.so
libcdc_vd_wmv3.so                  libawwmv3.so
libcdc_vencoder.so                 libvencoder.so
libcdc_ve.so                       libVE.so
libcdc_videoengine.so              libvideoengine.so

Description: The reason for modifying the name of the linux so is mainly to be consistent with the so name of the Android platform.

12). CedarC-v1.1.10

1. The changes are as follows：
1.1 Support android P system；
1.2 Compatible with Android P gms test；
1.3 Compatible with the reconstructed openmax module；
1.4 fix bugs.

2. Special note：

2.1 Open source the interface layer code of the encoder:
    Closed source: compile the entire encoder module into a so: libvendoer.so;
    Open source: Divide the encoder module into three submodules:libvendoer.so / libvenc_base.so / libvenc_codec.so;
          libvendoer.so / libvenc_base.so Two submodules are open source, libvenc_codec.so is the closed-source part.
          
　　Changes in usage: The applicable encoder only needs to link one so when it is closed source, and three sos need to be linked after it is open source.
12). CedarC-v1.2.0
1. The changes are as follows：
1.1 Add CdcMalloc.c, used to detect whether there is a leak in malloc's memory；
1.2 Add a sysfs mechanism, you can view sbm/fbm/vdecoder/memory and other information in real time through the cat ve node；
1.3 Add a dynamic debug mechanism by modifying the configuration file cedarc.conf implementation (after modifying the configuration file, you need to kill related processes such as mediaserver to take effect)；
1.4 The functional interface of the encoder isp is independent, and the user can call the VideoEncIspFunction interface separately.；
1.5 fix bugs.
