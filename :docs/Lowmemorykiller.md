Lowmemorykiller



​		在AMS的成员里，首先定义了一个变量final ProcessList mProcessList = new ProcessList( );当AMS初始化的时候，会将mProcessList也同时初始化，它的构造函数里调用了updateOomLevels方法，用于更新lowmemorykiller的参数，但是在AMS初始化的过程中，lowmemorykiller的参数并没有被写入到系统中，只有当AMS调用updateConfigurationXXX的时候才会

```java
private void updateOomLevels(int displayWidth, int displayHeight, boolean write) {
        // Scale buckets from avail memory: at 300MB we use the lowest values to
        // 700MB or more for the top values.
        float scaleMem = ((float)(mTotalMemMb-350))/(700-350);
        // Scale buckets from screen size.
        int minSize = 480*800;  //  384000
        int maxSize = 1280*800; // 1024000  230400 870400  .264
        float scaleDisp = ((float)(displayWidth*displayHeight)-minSize)/(maxSize-minSize);
				......
        float scale = scaleMem > scaleDisp ? scaleMem : scaleDisp;
        if (scale < 0) scale = 0;
        else if (scale > 1) scale = 1;
        int minfree_adj = Resources.getSystem().getInteger(
                com.android.internal.R.integer.config_lowMemoryKillerMinFreeKbytesAdjust);
        int minfree_abs = Resources.getSystem().getInteger(
                com.android.internal.R.integer.config_lowMemoryKillerMinFreeKbytesAbsolute);
   			......
        final boolean is64bit = Build.SUPPORTED_64_BIT_ABIS.length > 0;
        for (int i=0; i<mOomAdj.length; i++) {
            int low = mOomMinFreeLow[i];
            int high = mOomMinFreeHigh[i];
            if (is64bit) {
                // Increase the high min-free levels for cached processes for 64-bit
                if (i == 4) high = (high*3)/2;
                else if (i == 5) high = (high*7)/4;
            }
            mOomMinFree[i] = (int)(low + ((high-low)*scale));
        }
       ......
        }
			.....
        // Ask the kernel to try to keep enough memory free to allocate 3 full
        // screen 32bpp buffers without entering direct reclaim.
        int reserve = displayWidth * displayHeight * 4 * 3 / 1024;
				.......
        if (write) {
            ByteBuffer buf = ByteBuffer.allocate(4 * (2*mOomAdj.length + 1));
            buf.putInt(LMK_TARGET);
            for (int i=0; i<mOomAdj.length; i++) {
                buf.putInt((mOomMinFree[i]*1024)/PAGE_SIZE);
                buf.putInt(mOomAdj[i]);
            }
            writeLmkd(buf);
            SystemProperties.set("sys.sysctl.extra_free_kbytes", Integer.toString(reserve));
        }
        // GB: 2048,3072,4096,6144,7168,8192
        // HC: 8192,10240,12288,14336,16384,20480
    }

```

​		首先计算了一个scaleMem，当系统总内存大于700M，这个值就>1，否则<1。还有一个scaleDisp，当屏幕分辨率大于1280*800，这个就>1，否则<1。通过这两个值计算出一个scale，这个值在0～1之间。

​		ProcessList里预先定义了低端机的OOM各级别对应的memory数组mOomMinFreeLow和高端机对应的OOM各级别对应的memory数组mOomMinFreeHigh：

```java
    // These are the low-end OOM level limits.  This is appropriate for an
    // HVGA or smaller phone with less than 512MB.  Values are in KB.
    private final int[] mOomMinFreeLow = new int[] {
            12288, 18432, 24576,
            36864, 43008, 49152
    };
    // These are the high-end OOM level limits.  This is appropriate for a
    // 1280x800 or larger screen with around 1GB RAM.  Values are in KB.
    private final int[] mOomMinFreeHigh = new int[] {
            73728, 92160, 110592,
            129024, 147456, 184320
    };
```

​		通过这两个值可以计算最终的当前机器OOM各级别对应的memory数组mOomMinFree。

​		保留显示3个屏幕图像所需的内存，每一个像素用4个字节存储，所以每个屏幕图像所需的内存值是 displayWidth * displayHeight * 4，3个屏幕图像所需就是displayWidth * displayHeight * 4 * 3 / 1024，单位是kb。

​		计算出来的mOomMinFree通过socket写到系统中,写入的数值是==每个level的内存对应的page数==，而没有直接写内存数，内核中获取内存也是以page为单位；3个屏幕图像所需的值写入系统属性。

​	

