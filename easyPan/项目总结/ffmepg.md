## ffmepg基本用法

### 调用cmd执行ffmepg命令

```java
package com.easypan.utils;

import com.easypan.exception.BusinessException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;

public class ProcessUtils {
    private static final Logger logger = LoggerFactory.getLogger(ProcessUtils.class);

    public static String executeCommand(String cmd, Boolean outprintLog) throws BusinessException {
        if (StringTools.isEmpty(cmd)) {
            logger.error("--- 指令执行失败, 要执行的FFmpeg指令为空! ---");
            return null;
        }

        Runtime runtime = Runtime.getRuntime();
        Process process = null;
        try {
            process = Runtime.getRuntime().exec(cmd);
            // 执行ffmpeg指令
            // 取出输出流和错误流的信息
            // 注意: 必须取出ffmpeg在执行命令过程中产生的输出信息, 如果不取的话当输出信息填满jvm存储输出信息的缓冲区时, 线程就会回阻塞住
            PrintStream errorStream = new PrintStream(process.getErrorStream());
            PrintStream inputStream = new PrintStream(process.getInputStream());
            errorStream.start();
            inputStream.start();
            // 等待ffmpeg命令执行完
            process.waitFor();
            // 获取执行结果字符串
            String result = errorStream.stringBuffer.append(inputStream.stringBuffer + "\n").toString();
            // 输出执行的命令信息
            if (outprintLog) {
                logger.info("执行命令:{}, 已执行完毕,执行结果:{}", cmd, result);
            } else {
                logger.info("执行命令:{}, 已执行完毕", cmd);
            }
            return result;
        } catch (Exception e) {
            // logger.error("执行命令失败:{} ", e.getMessage());
            e.printStackTrace();
            throw new BusinessException("视频转换失败");
        } finally {
            if (null != process) {
                ProcessKiller ffmpegKiller = new ProcessKiller(process);
                runtime.addShutdownHook(ffmpegKiller);
            }
        }
    }

    /**
     * 在程序退出前结束已有的FFmpeg进程
     */
    private static class ProcessKiller extends Thread {
        private Process process;

        public ProcessKiller(Process process) { this.process = process; }

        @Override
        public void run() { this.process.destroy(); }
    }

    static class PrintStream extends Thread {
        InputStream inputStream = null;
        BufferedReader bufferedReader = null;
        StringBuffer stringBuffer = new StringBuffer();

        public PrintStream(InputStream inputStream) { this.inputStream = inputStream; }

        @Override
        public void run() {
            try {
                if (null == inputStream) {
                    return;
                }
                bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
                String line = null;
                while ((line = bufferedReader.readLine()) != null) {
                    stringBuffer.append(line);
                }
            } catch (Exception e) {
                logger.error("读取输入流出错了! 错误信息: " + e.getMessage());
            } finally {
                try {
                    if (null != bufferedReader) {
                        bufferedReader.close();
                    }
                    if (null != inputStream) {
                        inputStream.close();
                    }
                } catch (IOException e) {
                    logger.error("调用PrintStream读取输出流后, 关闭流时出错! ");
                }
            }
        }
    }
}
```

### 生成ts切面和m3u8索引文件

```java
private void curFile4Video(String fileId, String videoFilePath) {
        /**
         * 原始视频文件
         *     ↓
         * 转换为 TS 格式
         *     ↓
         * 切分成多个 TS 片段
         *     ↓
         * 生成 M3U8 索引文件
         *     ↓
         * 播放器读取 M3U8
         *     ↓
         * 依次请求 TS 片段
         */

        // 创建同名切片目录
        File tsFolder = new File(videoFilePath.substring(0, videoFilePath.lastIndexOf(".")));
        if (!tsFolder.exists()) {
            tsFolder.mkdirs();
        }
        /**
         * -y: 覆盖输出文件而不询问
         * -i %s: 输入文件
         * -vcodec copy: 复制视频流，不重新编码
         * -acodec copy: 复制音频流，不重新编码
         * -vbsf h264_mp4toannexb: 将 H.264 视频从 MP4 封装格式转换为 MPEG-2 TS 传输流格式
         * 这个命令的主要目的是将视频文件转换为 TS（Transport Stream）格式，同时保持原始的视频和音频质量（因为使用了 copy）。这种转换通常用于流媒体传输或者为后续切片做准备。
         */
        final String CMD_TRANSFER_2TS = "ffmpeg -y -i %s -vcodec copy -acodec copy -vbsf h264_mp4toannexb %s";

        /**
         * -i %s: 输入文件
         * -c copy: 复制所有流，不重新编码
         * -map 0: 从第一个输入文件选择所有流
         * -f segment: 启用分段输出格式
         * -segment_list %s: 指定生成分段列表文件的路径
         * -segment_time 30: 设置每个分段的时长为 30 秒
         * %s/%s_%%4d.ts: 输出文件的命名模式，其中 %%4d 表示用 4 位数字编号（例如：0001.ts, 0002.ts）
         * 这个命令的主要目的是将视频文件切分成多个 30 秒长的 TS 片段，
         */
        final String CMD_CUT_TS = "ffmpeg -i %s -c copy -map 0 -f segment -segment_list %s -segment_time 10 %s/%s_%%4d.ts";
        //  final String CMD_CUT_TS = "ffmpeg -i %s -c copy -map 0 -f segment -segment_list %s -segment_time 10 -hls_flags program_date_time %s/%s_%%4d.ts";

        String tsPath = tsFolder + "/" + Constants.TS_NAME;
        String cmd = String.format(CMD_TRANSFER_2TS, videoFilePath, tsPath);
        // 生成.ts
        ProcessUtils.executeCommand(cmd, false);
        // 生成索引文件.m3u8和切片.ts
        cmd = String.format(CMD_CUT_TS, tsPath, tsFolder.getPath() + "/" + Constants.M3U8_NAME, tsFolder.getPath(), fileId);
        ProcessUtils.executeCommand(cmd, false);
        // 删除index.ts
        new File(tsPath).delete();
    }
```

### 生成缩略图

```java
package com.easypan.utils;

import org.apache.commons.io.FileUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.File;

public class ScaleFilter {
    private static Logger logger = LoggerFactory.getLogger(ScaleFilter.class);

    /**
     * 利用ffmpeg生成视频封面
     * @param sourceFile 源文件
     * @param width 封面宽度
     * @param targetFile 目标文件
     */

    public static void createCover4Video(File sourceFile, Integer width, File targetFile) {
        try {
            String cmd = "ffmpeg -i %s -y -vframes 1 -vf scale=%d:%d/a %s";
            ProcessUtils.executeCommand(String.format(cmd, sourceFile.getAbsoluteFile(), width, width, targetFile.getAbsoluteFile()), false);
        } catch (Exception e) {
            logger.error("生成视频封面失败", e);
        }
    }

    /**
     * 创建缩略图
     */

    public static Boolean createThumbnailWidthFFmpeg(File file, int thumbnailWidth, File targetFile, Boolean delSource) {
        try {
            BufferedImage src = ImageIO.read(file);
            //thumbnailWidth 缩略图的宽度   thumbnailHeight 缩略图的高度
            int sorceW = src.getWidth();
            int sorceH = src.getHeight();
            //小于指定高宽不压缩
            if (sorceW <= thumbnailWidth) {
                return false;
            }
            compressImage(file, thumbnailWidth, targetFile, delSource);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * 利用ffmpeg进行图片压缩
     * @param sourceFile 需要压缩的源文件
     * @param width 压缩图片宽度
     * @param targetFile 压缩之后的目标文件
     * @param delSource 是否删除源文件
     */
    public static void compressImage(File sourceFile, Integer width, File targetFile, Boolean delSource) {
        try {
            String cmd = "ffmpeg -i %s -vf scale=%d:-1 %s -y";
            ProcessUtils.executeCommand(String.format(cmd, sourceFile.getAbsoluteFile(), width, targetFile.getAbsoluteFile()), false);
            if (delSource) {
                FileUtils.forceDelete(sourceFile);
            }
        } catch (Exception e) {
            logger.error("压缩图片失败", e);
        }
    }
}
```



