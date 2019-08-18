## iOS 语音对讲


### 关于音频队列


### AudioQueue
1.简述  
在iOS和Mac OS X中，音频队列是一个用来录制和播放音频的软件对象，他用AudioQueueRef这个不透明数据类型来表示，该类型在AudioQueue.h头文件中声明。
### 2.工作：  
- 连接音频硬件
- 内存管理
- 根据需要为已压缩的音频格式引入编码器
- 媒体的录制或播放

> 你可以将音频队列配合其他Core Audio的接口使用，再加上相对少量的自定义代码就可以在你的应用程序中创建一套完整的数字音频录制或播放解决方案。

### 3.结构：  
- 一组音频队列缓冲区(audio queue buffers)，每个音频队列缓冲区都是一个存储音频数据的临时仓库

- 一个缓冲区队列(buffer queue)，一个包含音频队列缓冲区的有序列表

- 一个你自己编写的音频队列回调函数(audio queue callback)

> 它的架构很大程度上依赖于这个音频队列是用来录制还是用来播放的。不同之处在于音频队列如何连接到它的输入和输入，还有它的回调函数所扮演的角色。


### 4.调用步骤  
首先将项目设置为MRC,在控制器中配置audioSession基本设置(基本设置，不会谷歌)，导入该头文件，直接在需要时机调用该类startRecord与stopRecord方法，另外还提供了生成录音文件的功能，具体参考github中的代码。
```
本例中涉及的一些宏定义,具体可以下载代码详细看
#define kBufferDurationSeconds              .5
#define kXDXRecoderAudioBytesPerPacket      2
#define kXDXRecoderAACFramesPerPacket       1024
#define kXDXRecoderPCMTotalPacket           512
#define kXDXRecoderPCMFramesPerPacket       1
#define kXDXRecoderConverterEncodeBitRate   64000
#define kXDXAudioSampleRate                 48000.0
```

### (1).设置AudioStreamBasicDescription 基本信息
```
-(void)startRecorder {
// Reset pcm_buffer to save convert handle, 每次开始音频会话前初始化pcm_buffer, pcm_buffer用来在捕捉声音的回调中存储累加的PCM原始数据
memset(pcm_buffer, 0, pcm_buffer_size);
pcm_buffer_size = 0;
frameCount      = 0;

// 是否正在录制
if (isRunning) {
// log4cplus_info("pcm", "Start recorder repeat");
return;
}

// 本例中采用log4打印log信息，若你没有可以不用，删除有关Log4的语句
// log4cplus_info("pcm", "starup PCM audio encoder");

// 设置采集的数据的类型为PCM
[self setUpRecoderWithFormatID:kAudioFormatLinearPCM];


OSStatus status          = 0;
UInt32   size            = sizeof(dataFormat);

// 编码器转码设置
[self convertBasicSetting];

// 这个if语句用来检测是否初始化本例对象成功,如果不成功重启三次,三次后如果失败可以进行其他处理
if (err != nil) {
NSString *error = nil;
for (int i = 0; i < 3; i++) {
usleep(100*1000);
error = [self convertBasicSetting];
if (error == nil) break;
}
// if init this class failed then restart three times , if failed again,can handle at there
//        [self exitWithErr:error];
}


// 新建一个队列,第二个参数注册回调函数，第三个防止内存泄露
status =  AudioQueueNewInput(&dataFormat, inputBufferHandler, (__bridge void *)(self), NULL, NULL, 0, &mQueue);
// log4cplus_info("pcm","AudioQueueNewInput status:%d",(int)status);

// 获取队列属性
status = AudioQueueGetProperty(mQueue, kAudioQueueProperty_StreamDescription, &dataFormat, &size);
// log4cplus_info("pcm","AudioQueueNewInput status:%u",(unsigned int)dataFormat.mFormatID);

// 这里将头信息添加到写入文件中，若文件数据为CBR,不需要添加，为VBR需要添加
[self copyEncoderCookieToFile];

//    可以计算获得，在这里使用的是固定大小
//    bufferByteSize = [self computeRecordBufferSizeFrom:&dataFormat andDuration:kBufferDurationSeconds];

// log4cplus_info("pcm","pcm raw data buff number:%d, channel number:%u",
kNumberQueueBuffers,
dataFormat.mChannelsPerFrame);

// 设置三个音频队列缓冲区
for (int i = 0; i != kNumberQueueBuffers; i++) {
// 注意：为每个缓冲区分配大小，可根据具体需求进行修改,但是一定要注意必须满足转换器的需求,转换器只有每次给1024帧数据才会完成一次转换,如果需求为采集数据量较少则用本例提供的pcm_buffer对数据进行累加后再处理
status = AudioQueueAllocateBuffer(mQueue, kXDXRecoderPCMTotalPacket*kXDXRecoderAudioBytesPerPacket*dataFormat.mChannelsPerFrame, &mBuffers[i]);
// 入队
status = AudioQueueEnqueueBuffer(mQueue, mBuffers[i], 0, NULL);
}

isRunning  = YES;
hostTime   = 0;

status     =  AudioQueueStart(mQueue, NULL);
log4cplus_info("pcm","AudioQueueStart status:%d",(int)status);
}
```

> 初始化输出流的结构体描述

```
struct AudioStreamBasicDescription
{
Float64          	mSampleRate;	    // 采样率 ：Hz
AudioFormatID      	mFormatID;	        // 采样数据的类型，PCM,AAC等
AudioFormatFlags    mFormatFlags;	    // 每种格式特定的标志，无损编码 ，0表示没有
UInt32            	mBytesPerPacket;    // 一个数据包中的字节数
UInt32              mFramesPerPacket;   // 一个数据包中的帧数，每个packet的帧数。如果是未压缩的音频数据，值是1。动态帧率格式，这个值是一个较大的固定数字，比如说AAC的1024。如果是动态大小帧数（比如Ogg格式）设置为0。
UInt32            	mBytesPerFrame;     // 每一帧中的字节数
UInt32            	mChannelsPerFrame;  // 每一帧数据中的通道数，单声道为1，立体声为2
UInt32              mBitsPerChannel;    // 每个通道中的位数，1byte = 8bit
UInt32              mReserved; 		    // 8字节对齐，填0
};
typedef struct AudioStreamBasicDescription  AudioStreamBasicDescription;

```

注意： kNumberQueueBuffers，音频队列可以使用任意数量的缓冲区。你的应用程序制定它的数量。一般情况下这个数字是3。这样就可以让给一个忙于将数据写入磁盘，同时另一个在填充新的音频数据，第三个缓冲区在需要做磁盘I/O延迟补偿的时候可用


> 如何使用AudioQueue:

1. 创建输入队列AudioQueueNewInput
2. 分配buffers
3. 入队：AudioQueueEnqueueBuffer
4. 回调函数采集音频数据
5. 出队

> AudioQueueNewInput

```
// 作用：创建一个音频队列为了录制音频数据
原型：extern OSStatus             
AudioQueueNewInput( const AudioStreamBasicDescription   *inFormat, 同上
AudioQueueInputCallback             inCallbackProc, // 注册回调函数
void * __nullable               	inUserData,		
CFRunLoopRef __nullable         	inCallbackRunLoop,
CFStringRef __nullable          	inCallbackRunLoopMode,
UInt32                          	inFlags,
AudioQueueRef __nullable        	* __nonnull outAQ)；

// 这个函数的第四个和第五个参数是有关于线程的，我设置成null，代表它默认使用内部线程去录音，而且还是异步的
```

### (2).设置采集数据的格式，采集PCM必须按照如下设置，参考苹果官方文档，不同需求自己另行修改

```
-(void)setUpRecoderWithFormatID:(UInt32)formatID {
// Notice : The settings here are official recommended settings,can be changed according to specific requirements. 此处的设置为官方推荐设置,可根据具体需求修改部分设置
//setup auido sample rate, channel number, and format ID
memset(&dataFormat, 0, sizeof(dataFormat));

UInt32 size = sizeof(dataFormat.mSampleRate);
AudioSessionGetProperty(kAudioSessionProperty_CurrentHardwareSampleRate,
&size,
&dataFormat.mSampleRate);
dataFormat.mSampleRate = kXDXAudioSampleRate; // 设置采样率

size = sizeof(dataFormat.mChannelsPerFrame);
AudioSessionGetProperty(kAudioSessionProperty_CurrentHardwareInputNumberChannels,
&size,
&dataFormat.mChannelsPerFrame);
dataFormat.mFormatID = formatID;

// 关于采集PCM数据是根据苹果官方文档给出的Demo设置，至于为什么这么设置可能与采集回调函数内部实现有关，修改的话请谨慎
if (formatID == kAudioFormatLinearPCM)
{
/*
为保存音频数据的方式的说明，如可以根据大端字节序或小端字节序，
浮点数或整数以及不同体位去保存数据
例如对PCM格式通常我们如下设置：kAudioFormatFlagIsSignedInteger | kAudioFormatFlagIsPacked等
*/
dataFormat.mFormatFlags     = kLinearPCMFormatFlagIsSignedInteger | kLinearPCMFormatFlagIsPacked;
// 每个通道里，一帧采集的bit数目
dataFormat.mBitsPerChannel  = 16;
// 8bit为1byte，即为1个通道里1帧需要采集2byte数据，再*通道数，即为所有通道采集的byte数目
dataFormat.mBytesPerPacket  = dataFormat.mBytesPerFrame = (dataFormat.mBitsPerChannel / 8) * dataFormat.mChannelsPerFrame;
// 每个包中的帧数，采集PCM数据需要将dataFormat.mFramesPerPacket设置为1，否则回调不成功
dataFormat.mFramesPerPacket = kXDXRecoderPCMFramesPerPacket;
}
}
```

### (3).将PCM转成AAC一些基本设置

```
-(NSString *)convertBasicSetting {
// 此处目标格式其他参数均为默认，系统会自动计算，否则无法进入encodeConverterComplexInputDataProc回调

AudioStreamBasicDescription sourceDes = dataFormat; // 原始格式
AudioStreamBasicDescription targetDes;              // 转码后格式

// 设置目标格式及基本信息
memset(&targetDes, 0, sizeof(targetDes));
targetDes.mFormatID           = kAudioFormatMPEG4AAC;
targetDes.mSampleRate         = kXDXAudioSampleRate;
targetDes.mChannelsPerFrame   = dataFormat.mChannelsPerFrame;
targetDes.mFramesPerPacket    = kXDXRecoderAACFramesPerPacket; // 采集的为AAC需要将targetDes.mFramesPerPacket设置为1024，AAC软编码需要喂给转换器1024个样点才开始编码，这与回调函数中inNumPackets有关，不可随意更改

OSStatus status     = 0;
UInt32 targetSize   = sizeof(targetDes);
status              = AudioFormatGetProperty(kAudioFormatProperty_FormatInfo, 0, NULL, &targetSize, &targetDes);
// log4cplus_info("pcm", "create target data format status:%d",(int)status);

memset(&_targetDes, 0, sizeof(_targetDes));
// 赋给全局变量
memcpy(&_targetDes, &targetDes, targetSize);

// 选择软件编码
AudioClassDescription audioClassDes;
status = AudioFormatGetPropertyInfo(kAudioFormatProperty_Encoders,
sizeof(targetDes.mFormatID),
&targetDes.mFormatID,
&targetSize);
// log4cplus_info("pcm","get kAudioFormatProperty_Encoders status:%d",(int)status);

// 计算编码器容量
UInt32 numEncoders = targetSize/sizeof(AudioClassDescription);
// 用数组存放编码器内容
AudioClassDescription audioClassArr[numEncoders];
// 将编码器属性赋给数组
AudioFormatGetProperty(kAudioFormatProperty_Encoders,
sizeof(targetDes.mFormatID),
&targetDes.mFormatID,
&targetSize,
audioClassArr);
// log4cplus_info("pcm","wrirte audioClassArr status:%d",(int)status);

// 遍历数组，设置软编
for (int i = 0; i < numEncoders; i++) {
if (audioClassArr[i].mSubType == kAudioFormatMPEG4AAC && audioClassArr[i].mManufacturer == kAppleSoftwareAudioCodecManufacturer) {
memcpy(&audioClassDes, &audioClassArr[i], sizeof(AudioClassDescription));
break;
}
}

// 防止内存泄露	
if (_encodeConvertRef == NULL) {
// 新建一个编码对象，设置原，目标格式
status          = AudioConverterNewSpecific(&sourceDes, &targetDes, 1,
&audioClassDes, &_encodeConvertRef);

if (status != noErr) {
//            log4cplus_info("Audio Recoder","new convertRef failed status:%d \n",(int)status);
return @"Error : New convertRef failed \n";
}
}    

// 获取原始格式大小
targetSize      = sizeof(sourceDes);
status          = AudioConverterGetProperty(_encodeConvertRef, kAudioConverterCurrentInputStreamDescription, &targetSize, &sourceDes);
// log4cplus_info("pcm","get sourceDes status:%d",(int)status);

// 获取目标格式大小
targetSize      = sizeof(targetDes);
status          = AudioConverterGetProperty(_encodeConvertRef, kAudioConverterCurrentOutputStreamDescription, &targetSize, &targetDes);;
// log4cplus_info("pcm","get targetDes status:%d",(int)status);

// 设置码率，需要和采样率对应
UInt32 bitRate  = kXDXRecoderConverterEncodeBitRate;
targetSize      = sizeof(bitRate);
status          = AudioConverterSetProperty(_encodeConvertRef,
kAudioConverterEncodeBitRate,
targetSize, &bitRate);
// log4cplus_info("pcm","set covert property bit rate status:%d",(int)status);
if (status != noErr) {
//        log4cplus_info("Audio Recoder","set covert property bit rate status:%d",(int)status);
return @"Error : Set covert property bit rate failed";
}

return nil;

}
```

> AudioFormatGetProperty：

```
原型： 
extern OSStatus

AudioFormatGetProperty(	AudioFormatPropertyID    inPropertyID,
UInt32				        inSpecifierSize,
const void * __nullable  inSpecifier,
UInt32 	 * __nullable  ioPropertyDataSize,
void * __nullabl         outPropertyData);
作用：检索某个属性的值
```

> AudioClassDescription：

指的是一个能够对一个信号或者一个数据流进行变换的设备或者程序。这里指的变换既包括将 信号或者数据流进行编码（通常是为了传输、存储或者加密）或者提取得到一个编码流的操作，也包括为了观察或者处理从这个编码流中恢复适合观察或操作的形式的操作。编解码器经常用在视频会议和流媒体等应用中。


默认情况下，Apple会创建一个硬件编码器，如果硬件不可用，会创建软件编码器。

经过我的测试，硬件AAC编码器的编码时延很高，需要buffer大约2秒的数据才会开始编码。而软件编码器的编码时延就是正常的，只要喂给1024个样点，就会开始编码。

```
AudioConverterNewSpecific：
原型： extern OSStatus
AudioConverterNewSpecific(  const AudioStreamBasicDescription * inSourceFormat,
const AudioStreamBasicDescription * inDestinationFormat,
UInt32                              inNumberClassDescriptions,
const AudioClassDescription *       inClassDescriptions,
AudioConverterRef __nullable * __nonnull outAudioConverter)；

解释：创建一个转换器
作用：设置一些转码基本信息          
```

```
AudioConverterSetProperty：
原型：extern OSStatus 
AudioConverterSetProperty(  AudioConverterRef           inAudioConverter,
AudioConverterPropertyID    inPropertyID,
UInt32                      inPropertyDataSize,
const void *                inPropertyData)；
作用：设置码率，需要注意，AAC并不是随便的码率都可以支持。比如如果PCM采样率是44100KHz，那么码率可以设置64000bps，如果是16K，可以设置为32000bps。
```

### (4).设置最终音频文件的头部信息(此类写法为将pcm转为AAC的写法)

```
-(void)copyEncoderCookieToFile
{
// Grab the cookie from the converter and write it to the destination file.
UInt32 cookieSize = 0;
OSStatus error = AudioConverterGetPropertyInfo(_encodeConvertRef, kAudioConverterCompressionMagicCookie, &cookieSize, NULL);

// If there is an error here, then the format doesn't have a cookie - this is perfectly fine as som formats do not.
// log4cplus_info("cookie","cookie status:%d %d",(int)error, cookieSize);
if (error == noErr && cookieSize != 0) {
char *cookie = (char *)malloc(cookieSize * sizeof(char));
//        UInt32 *cookie = (UInt32 *)malloc(cookieSize * sizeof(UInt32));
error = AudioConverterGetProperty(_encodeConvertRef, kAudioConverterCompressionMagicCookie, &cookieSize, cookie);
// log4cplus_info("cookie","cookie size status:%d",(int)error);

if (error == noErr) {
error = AudioFileSetProperty(mRecordFile, kAudioFilePropertyMagicCookieData, cookieSize, cookie);
// log4cplus_info("cookie","set cookie status:%d ",(int)error);
if (error == noErr) {
UInt32 willEatTheCookie = false;
error = AudioFileGetPropertyInfo(mRecordFile, kAudioFilePropertyMagicCookieData, NULL, &willEatTheCookie);
printf("Writing magic cookie to destination file: %u\n   cookie:%d \n", (unsigned int)cookieSize, willEatTheCookie);
} else {
printf("Even though some formats have cookies, some files don't take them and that's OK\n");
}
} else {
printf("Could not Get kAudioConverterCompressionMagicCookie from Audio Converter!\n");
}

free(cookie);
}
}
```

> Magic cookie 是一种不透明的数据格式，它和压缩数据文件与流联系密切，如果文件数据为CBR格式（无损），则不需要添加头部信息，如果为VBR需要添加,// if collect CBR needn't set magic cookie , if collect VBR should set magic cookie, if needn't to convert format that can be setting by audio queue directly.

### (5).AudioQueue中注册的回调函数

```
// AudioQueue中注册的回调函数
static void inputBufferHandler(void *                                 inUserData,
AudioQueueRef                          inAQ,
AudioQueueBufferRef                    inBuffer,
const AudioTimeStamp *                 inStartTime,
UInt32                                 inNumPackets,
const AudioStreamPacketDescription*	  inPacketDesc) {
// 相当于本类对象实例
TVURecorder *recoder        = (TVURecorder *)inUserData;

/*
inNumPackets 总包数：音频队列缓冲区大小 （在先前估算缓存区大小为kXDXRecoderAACFramesPerPacket*2）/ （dataFormat.mFramesPerPacket (采集数据每个包中有多少帧，此处在初始化设置中为1) * dataFormat.mBytesPerFrame（每一帧中有多少个字节，此处在初始化设置中为每一帧中两个字节）），所以可以根据该公式计算捕捉PCM数据时inNumPackets。
注意：如果采集的数据是PCM需要将dataFormat.mFramesPerPacket设置为1，而本例中最终要的数据为AAC,因为本例中使用的转换器只有每次传入1024帧才能开始工作,所以在AAC格式下需要将mFramesPerPacket设置为1024.也就是采集到的inNumPackets为1，在转换器中传入的inNumPackets应该为AAC格式下默认的1，在此后写入文件中也应该传的是转换好的inNumPackets,如果有特殊需求需要将采集的数据量小于1024,那么需要将每次捕捉到的数据先预先存储在一个buffer中,等到攒够1024帧再进行转换。
*/

// collect pcm data，可以在此存储

// First case : collect data not is 1024 frame, if collect data not is 1024 frame, we need to save data to pcm_buffer untill 1024 frame
memcpy(pcm_buffer+pcm_buffer_size, inBuffer->mAudioData, inBuffer->mAudioDataByteSize);
pcm_buffer_size = pcm_buffer_size + inBuffer->mAudioDataByteSize;
if(inBuffer->mAudioDataByteSize != kXDXRecoderAACFramesPerPacket*2)
NSLog(@"write pcm buffer size:%d, totoal buff size:%d", inBuffer->mAudioDataByteSize, pcm_buffer_size);

frameCount++;

// Second case : If the size of the data collection is not required, we can let mic collect 1024 frame so that don't need to write firtst case, but it is recommended to write the above code because of agility 

// if collect data is added to 1024 frame
if(frameCount == totalFrames) {
AudioBufferList *bufferList = convertPCMToAAC(recoder);
pcm_buffer_size = 0;
frameCount      = 0;

// free memory
free(bufferList->mBuffers[0].mData);
free(bufferList);
// begin write audio data for record audio only

// 出队
AudioQueueRef queue = recoder.mQueue;
if (recoder.isRunning) {
AudioQueueEnqueueBuffer(queue, inBuffer, 0, NULL);
}
}
}
```
> 解析回调函数：相当于中断服务函数，每次录取到音频数据就进入这个函数  

注意：inNumPackets 总包数：音频队列缓冲区大小 （在先前估算缓存区大小为2048）/ （dataFormat.mFramesPerPacket (采集数据每个包中有多少帧，此处在初始化设置中为1) * dataFormat.mBytesPerFrame（每一帧中有多少个字节，此处在初始化设置中为每一帧中两个字节））

- inAQ 是调用回调函数的音频队列  
- inBuffer 是一个被音频队列填充新的音频数据的音频队列缓冲区，它包含了回调函数写入文件所需要的新数据  
- inStartTime 是缓冲区中的一采样的参考时间，对于基本的录制，你的毁掉函数不会使用这个参数  
- inNumPackets是inPacketDescs参数中包描述符（packet descriptions）的数量，如果你正在录制一个VBR(可变比特率（variable bitrate））格式, 音频队列将会提供这个参数给你的回调函数，这个参数可以让你传递给AudioFileWritePackets函数. CBR (常量比特率（constant bitrate）) 格式不使用包描述符。对于CBR录制，音频队列会设置这个参数并且将inPacketDescs这个参数设置为NULL，官方解释为The number of packets of audio data sent to the callback in the inBuffer parameter.

```
// PCM -> AAC
AudioBufferList* convertPCMToAAC (AudioQueueBufferRef inBuffer, XDXRecorder *recoder) {

UInt32   maxPacketSize    = 0;
UInt32   size             = sizeof(maxPacketSize);
OSStatus status;

status = AudioConverterGetProperty(_encodeConvertRef,
kAudioConverterPropertyMaximumOutputPacketSize,
&size,
&maxPacketSize);
// log4cplus_info("AudioConverter","kAudioConverterPropertyMaximumOutputPacketSize status:%d \n",(int)status);

// 初始化一个bufferList存储数据
AudioBufferList *bufferList             = (AudioBufferList *)malloc(sizeof(AudioBufferList));
bufferList->mNumberBuffers              = 1;
bufferList->mBuffers[0].mNumberChannels = _targetDes.mChannelsPerFrame;
bufferList->mBuffers[0].mData           = malloc(maxPacketSize);
bufferList->mBuffers[0].mDataByteSize   = pcm_buffer_size;

AudioStreamPacketDescription outputPacketDescriptions;

/*     
inNumPackets设置为1表示编码产生1帧数据即返回，官方：On entry, the capacity of outOutputData expressed in packets in the converter's output format. On exit, the number of packets of converted data that were written to outOutputData. 在输入表示输出数据的最大容纳能力 在转换器的输出格式上，在转换完成时表示多少个包被写入
*/
UInt32 inNumPackets = 1;
status = AudioConverterFillComplexBuffer(_encodeConvertRef,
encodeConverterComplexInputDataProc,	// 填充数据的回调函数
pcm_buffer,		// 音频队列缓冲区中数据
&inNumPackets,		
bufferList,			// 成功后将值赋给bufferList
&outputPacketDescriptions);	// 输出包包含的一些信息
log4cplus_info("AudioConverter","set AudioConverterFillComplexBuffer status:%d",(int)status);

if (recoder.needsVoiceDemo) {
// if inNumPackets set not correct, file will not normally play. 将转换器转换出来的包写入文件中，inNumPackets表示写入文件的起始位置
OSStatus status = AudioFileWritePackets(recoder.mRecordFile,
FALSE,
bufferList->mBuffers[0].mDataByteSize,
&outputPacketDescriptions,
recoder.mRecordPacket,
&inNumPackets,
bufferList->mBuffers[0].mData);
//        log4cplus_info("write file","write file status = %d",(int)status);
recoder.mRecordPacket += inNumPackets;  // Used to record the location of the write file,用于记录写入文件的位置
}

return bufferList;
}
```
> 解析

outputPacketDescriptions数组是每次转换的AAC编码后各个包的描述,但这里每次只转换一包数据(由传入的packetSize决定)。调用AudioConverterFillComplexBuffer触发转码，他的第二个参数是填充原始音频数据的回调。转码完成后，会将转码的数据存放在它的第五个参数中(bufferList).

```
// 录制声音功能
-(void)startVoiceDemo
{
NSArray *searchPaths    = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
NSString *documentPath  = [[searchPaths objectAtIndex:0] stringByAppendingPathComponent:@"VoiceDemo"];
OSStatus status;

// Get the full path to our file.
NSString *fullFileName  = [NSString stringWithFormat:@"%@.%@",[[XDXDateTool shareXDXDateTool] getDateWithFormat_yyyy_MM_dd_HH_mm_ss],@"caf"];
NSString *filePath      = [documentPath stringByAppendingPathComponent:fullFileName];
[mRecordFilePath release];
mRecordFilePath         = [filePath copy];;
CFURLRef url            = CFURLCreateWithString(kCFAllocatorDefault, (CFStringRef)filePath, NULL);

// create the audio file
status                  = AudioFileCreateWithURL(url, kAudioFileMPEG4Type, &_targetDes, kAudioFileFlags_EraseFile, &mRecordFile);
if (status != noErr) {
// log4cplus_info("Audio Recoder","AudioFileCreateWithURL Failed, status:%d",(int)status);
}

CFRelease(url);

// add magic cookie contain header file info for VBR data
[self copyEncoderCookieToFile];

mNeedsVoiceDemo         = YES;
NSLog(@"%s",__FUNCTION__);
}
```











### 参考
[iOS语音对讲（一）实时采集PCM并编码AAC](https://www.jianshu.com/p/b89b6be8fb04)       
[iOS音频编程之实时语音通信（对讲机功能）](https://www.jianshu.com/p/781ccf6f80d3)  

[1小时学会：最简单的iOS直播推流（五）yuv、pcm数据的介绍和获取](https://blog.csdn.net/hard_man/article/details/53193046)  


//   
[iOS AudioQueue实时录音](https://www.jianshu.com/p/84a0df439431)  
[iOS 实时录音和播放](https://blog.csdn.net/hucuiyun/article/details/78234042)  
[音频队列服务编程指南(Audio Queue Services Programming Guide)(二)](https://blog.csdn.net/jiangyiaxiu/article/details/9190035)  
[iOS学习笔记2-使用Audio Queues录音，取得实时PCM数据](https://blog.csdn.net/xiaoluodecai/article/details/47153945)  
[AudioQueue 播放和录音](https://blog.csdn.net/lyy_whg/article/details/9919785)  
[iOS音频播放 (五)：AudioQueue 转](https://blog.csdn.net/sqc3375177/article/details/38532207)  
[iOS音频学习三之AudioQueue](https://www.jianshu.com/p/62b584540672)  

[iOS播放PCM,NSData流代码（Audio Queue Services）](https://segmentfault.com/a/1190000010177336)  
[ios利用mic采集Pcm转为AAC，AudioQueue、AudioUnit(流式)](https://www.jianshu.com/p/e2d072b9e4d8)  



[iOS音频掌柜-- AVAudioSession](https://www.jianshu.com/p/3e0a399380df/)  
[iOS实时录音](https://www.jianshu.com/p/2556022786de)  
[【iOS录音与播放】实现利用音频队列，通过缓存进行对声音的采集与播放](http://www.cnblogs.com/anjohnlv/p/3383908.html)  
[iOS 实时录音和播放](http://www.cnblogs.com/CoderEYLee/p/Object-C-0026.html)  

[iOS音频系列(三)--AudioQueue](https://www.jianshu.com/p/5d165bf2233d)  
[iOS AudioQueue实时录音](https://www.jianshu.com/p/84a0df439431)  
[AudioQueue 播放和录音](https://blog.csdn.net/lyy_whg/article/details/9919785)  


[ios利用mic采集Pcm转为AAC，AudioQueue、AudioUnit(流式),audioqueue](http://www.code4app.com/thread-27323-1-1.html)  
[ios利用mic采集Pcm转为AAC，AudioQueue、AudioUnit(流式)](https://www.jianshu.com/p/e2d072b9e4d8)  
[音频队列服务编程指南(Audio Queue Services Programming Guide)(三)](https://blog.csdn.net/just_we_0727/article/details/8373440?utm_source=blogxgwz7)  

[Using Audio Queue -- Recorder -- (Part 1)](https://github.com/lancy/Octopress/blob/8cc59c2c118110d85c730e061073c29f979c319a/source/_posts/using_audio_queue.md)

[【音频】G711编码原理](https://blog.csdn.net/q2519008/article/details/80900838)    
[iOS AVAudioRecorder录音,转MP3,播放](https://www.jianshu.com/p/72887a79e949)  


[audioqueue 播放以及录制pcm数据](https://github.com/xixi9527/AudioqueuePlayer)  
[XYRealTimeRecord](https://github.com/wszxy/XYRealTimeRecord)  
[audioqueue 播放以及录制pcm数据](https://github.com/xixi9527/AudioqueuePlayer)  


[音频队列服务编程指南(Audio Queue Services Programming Guide)(二)](https://blog.csdn.net/jiangyiaxiu/article/details/9190035)  

[AudioQueueServiceDemo](https://github.com/brownfeng/AudioQueueServiceDemo)  


[Music147_2012](https://github.com/kumezaki/Music147_2012)  



