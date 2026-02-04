---
title: CentOS7 + Selenium + Tess4j
date: "2026-02-04T10:23:00.000Z"
---

记个流水账

## Selenium

由于CentOS7上面的库太老了，不支持最新的Chrome版本（比如144），最新也只能是114版本。而且由于Ali服务器不能连接google网站，只能是手工下载，然后上传。根据[guide](https://developer.chrome.com/docs/chromedriver/downloads?hl=zh-cn#chromedriver_1140573590)首先选定版本为`114.0.5735.90`
下面开始下载和安装，必须安装下面两个：
### 下载
* ChromeDriver 
根据[版本选择](https://developer.chrome.com/docs/chromedriver/downloads/version-selection?hl=zh-cn)，找到下载[地址](https://chromedriver.storage.googleapis.com/index.html?path=114.0.5735.90/), 选择chromedriver_linux64.zip下载到本地然后上传到服务器

* Chrome
可以根据google[Json文件]的指引，直接这个[地址](https://storage.googleapis.com/chrome-for-testing-public/114.0.5735.90/linux64/chrome-linux64.zip)下载就是了。

需要说明一点的是以上两个文件版本必须严格一致。另外对于115以上的版本，ChromeDriver就可以和Chrome一样下载，不需要到另外一个网站下载了。

### 安装

* 安装Chrome
```bash
sudo yum install -y unzip # If unzip is not installed
sudo unzip chrome-linux64.zip -d /opt/
# Replace 'chrome-linux64/chrome' with the actual path inside your unzipped folder
sudo ln -s /opt/chrome-linux64/chrome /usr/bin/google-chrome
sudo yum install -y libX11 libXcomposite libXcursor libXdamage libXext libXi libXrandr libXscrnsaver libXtst pango at-spi2-atk libXt alsa-lib atk cups-libs gtk3
chmod +x /opt/chrome-linux64/chrome
```

然后验证一下
```bash
ldd /opt/chrome-linux64/chrome | grep "not found"
google-chrome --version
```

* 安装ChromeDriver

这个不需要安装，只需要把下载的文件解压并放在你指定的位置即可。
```bash
chmod +x /opt/app/lib/chromedriver
```

```java
String chromeDir = System.getProperty("os.name").toLowerCase().contains("win")
        ? "offlinetransfer/lib/chromedriver.exe"
        : "/opt/app/lib/chromedriver";

System.setProperty("webdriver.chrome.driver", chromeDir);

options.addArguments("--headless=new");
options.addArguments("--window-size=1024,768");
options.addArguments("--disable-blink-features=AutomationControlled");
options.addArguments("--user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36");

// For Linux Headless, these are mandatory
if (!System.getProperty("os.name").toLowerCase().contains("win")) {
    options.addArguments("--no-sandbox"); // Critical for Linux root/docker
    options.addArguments("--disable-dev-shm-usage"); // Prevents memory crashes in containers
}        
```

## Tess4j

在Windows上运行的时候和CentOS上运行非常不同，在Windows上面运行很简单，只需要提供训练数据包并提供文件地址即可。但是在CentOS7上，需要先安装运行库(.so文件)，而且训练数据包必须放在一个文件夹里，而你指定的文件夹必须是这个文件夹的上一级。下面的例子中，对于windows来说，`en.traineddata`就放在`/lib/`下面即可，而在CentOS7上，你必须放在`/usr/share/tesseract/tessdata/`

```java
tesseract = new Tesseract();
String baseDir = System.getProperty("os.name").toLowerCase().contains("win")
        ? "offlinetransfer/lib/"
        : "/usr/share/tesseract";
tesseract.setDatapath(baseDir);
tesseract.setLanguage("eng");
```
* 安装Tesseract

```bash
sudo yum install -y epel-release
sudo yum install -y tesseract tesseract-devel

#Verify
ldconfig -p | grep tesseract 
```

完成这个安装后，对应版本的训练数据也安装好了，千万不要自己再手工下载，这样可能会导致版本不一致。在CentOS7上，对应的版本应该是 Tess4J 4.5.4 + Tesseract 3.04.0



